## 功能背景
保持整体内容框架 但是可以描述得更加详细一点。
## 重大问题分析
保持整体内容框架不变 但可以描述得更加详细一点。
## 方案设计
对于双循环架构,这里面额外再补充一下我们多加了个状态表示是cur_folio_in_commpress_ctx。
这是我们的补充。在这种状态下的folio也不应该被解锁。
然后我之前的这份文档里没给出f2fs_do_read_multi_folios函数的代码分析
,现在我将他们给给出来。
你增量改动的话首先是详细地阐述这几个核心函数的代码。
重点将增加read_bytes_pending的地方给标为红色。忽略掉我用于调试打印的那些代码。
```C
__attribute__((optimize("O0"))) 
int f2fs_do_read_multi_folios(struct f2fs_readpage_ctx* ctx, loff_t pos,loff_t plen)
{
	struct folio *folio = ctx->cur_folio;
	struct compress_ctx*cc=&ctx->cc;
	struct bio** bio_ret = &ctx->bio;
	pgoff_t cur_idx = pos >> PAGE_SHIFT;
	struct readahead_control *rac = ctx->rac;
	int ret = 0;	
	pgoff_t cluster_ix = cluster_idx(cc,cur_idx);
	if (cc->cluster_idx == NULL_CLUSTER)
	{
		ret = f2fs_init_compress_ctx(cc);
		if (ret < 0)
			return ret;
	}
	ctx->cur_folio_in_compress_ctx = true;
	ret =do_read_multi_folios(cc, folio, pos, plen,bio_ret,rac,false);
	return ret;
}
int do_read_multi_folios(struct compress_ctx*cc, struct folio *folio, loff_t pos,
				  loff_t plen, struct bio** bio_ret,
				  struct readahead_control *rac,bool for_write)
{ 
	int ret = 0;
	f2fs_compress_ctx_add_folio(cc, folio, pos,plen);
	struct f2fs_iomap_folio_state* fifs = folio->private;
	if(folio_order(folio)>0&&fifs)
	{
		spin_lock_irq(&fifs->state_lock);
		fifs->read_bytes_pending += plen;
		spin_unlock_irq(&fifs->state_lock);
	}
	#ifdef CONFIG_F2FS_DEBUG_PRINT
	f2fs_err(F2FS_F_SB(folio),"after rbp add:");
	FUNC(print_rbp,folio);
	#endif
	/* No more flushing compress ctx rpages and cpages to io
	when a new folio's cluster index change,but flush immediately when cc
	is full or this is the last folio in rac*/
	if (f2fs_cluster_is_full(cc) || (rac&&rac->_nr_pages == 0)) {
		ret = f2fs_read_multi_folios(cc, bio_ret,rac,for_write);
		f2fs_destroy_compress_ctx(cc, false);
	}
	return ret;
}
```
我发现唯一一处需要真的就是覆盖性改动的是,原先的ai,对我的f2fs_compress_ctx_add_folio,它私自的改了我的定义。这让我很生气。
你一定要遵守我们一直以来所说的,在我的要求下,该增量修改的增量修改,我明确要求覆盖改的再改。
我给出f2fs_compress_ctx_add_folio的定义:
```C
/*Add part of folio into compress_ctx*/
__attribute__((optimize("O0")))
void f2fs_compress_ctx_add_folio(struct compress_ctx *cc, struct folio *folio,
				 loff_t pos, loff_t rlen)
{	
	unsigned int cluster_ofs;
	loff_t poff = offset_in_folio(folio, pos);
	unsigned int idx_in_folio = poff >> PAGE_SHIFT;
	if (!f2fs_cluster_can_merge_page(cc, folio->index+idx_in_folio))
		f2fs_bug_on(F2FS_I_SB(cc->inode), 1);
	cluster_ofs = offset_in_cluster(cc, folio->index+idx_in_folio);
	int num_pages_to_add =
		min_t(unsigned int,(unsigned int)folio_nr_pages(folio) - idx_in_folio,
		    cc->cluster_size - cc->nr_rpages);
	num_pages_to_add = min_t(unsigned int,num_pages_to_add, rlen >> PAGE_SHIFT);
	for (int i = 0; i < num_pages_to_add; ++i) {
		cc->rpages[cluster_ofs] = folio_page(folio, idx_in_folio + i);
		cc->nr_rpages++;
		cluster_ofs++;
	}
	cc->cluster_idx = cluster_idx(cc, folio->index+idx_in_folio);
}
```
然后我原先的文案里,光顾着说让read_bytes_pending增加的事情了,实际上还有read_bytes_pending,需要进行减少的部分我没加上去。
我将这部分代码的定义也放出来。
首先f2fs读io结束的直接回调函数是
```C
static void f2fs_read_end_io(struct bio *bio)
{
	struct f2fs_sb_info *sbi = F2FS_P_SB(bio_first_page_all(bio));
	struct bio_post_read_ctx *ctx;
	bool intask = in_task();

	iostat_update_and_unbind_ctx(bio);
	ctx = bio->bi_private;

	if (time_to_inject(sbi, FAULT_READ_IO))
		bio->bi_status = BLK_STS_IOERR;

	if (bio->bi_status) {
		f2fs_finish_read_bio(bio, intask);
		return;
	}

	if (ctx) {
		unsigned int enabled_steps = ctx->enabled_steps &
					(STEP_DECRYPT | STEP_DECOMPRESS);

		/*
		 * If we have only decompression step between decompression and
		 * decrypt, we don't need post processing for this.
		 */
		if (enabled_steps == STEP_DECOMPRESS &&
				!f2fs_low_mem_mode(sbi)) {
			f2fs_handle_step_decompress(ctx, intask);
		} else if (enabled_steps) {
			INIT_WORK(&ctx->work, f2fs_post_read_work);
			queue_work(ctx->sbi->post_read_wq, &ctx->work);
			return;
		}
	}

	f2fs_verify_and_finish_bio(bio, intask);
}
```
普通页的真正的结束路径是里面的
```C
static void f2fs_finish_read_bio(struct bio *bio, bool in_task)
{
	struct folio_iter fi;
	struct bio_post_read_ctx *ctx = bio->bi_private;

	bio_for_each_folio_all(fi, bio) {
		struct folio *folio = fi.folio;

		if (f2fs_is_compressed_folio(folio)) {
			if (ctx && !ctx->decompression_attempted)
				f2fs_end_read_compressed_page(&folio->page, true, 0,
							in_task);
			f2fs_put_folio_dic(folio, in_task);
			continue;
		}

		dec_page_count(F2FS_F_SB(folio), __read_io_type(folio));
		int error=!(bio->bi_status == 0);
		f2fs_iomap_finish_folio_read(folio,fi.offset,
		fi.length,error);
	}

	if (ctx)
		mempool_free(ctx, bio_post_read_ctx_pool);
	bio_put(bio);
}
```
而针对压缩文件的路径则是
```C
__attribute__((optimize("O0")))
void f2fs_decompress_end_io(struct decompress_io_ctx *dic, bool failed,
				bool in_task)
{
	int i;

	if (!failed && dic->need_verity) {
		/*
		 * Note that to avoid deadlocks, the verity work can't be done
		 * on the decompression workqueue.  This is because verifying
		 * the data pages can involve reading metadata pages from the
		 * file, and these metadata pages may be compressed.
		 */
		INIT_WORK(&dic->verity_work, f2fs_verify_cluster);
		fsverity_enqueue_verify_work(&dic->verity_work);
		return;
	}

	/* Update and unlock the cluster's pagecache pages. */
	// for (i = 0; i < dic->cluster_size; i++) {
	// 	struct page *rpage = dic->rpages[i];

	// 	if (!rpage)
	// 		continue;

	// 	if (failed)
	// 		ClearPageUptodate(rpage);
	// 	else
	// 		SetPageUptodate(rpage);
	// 	unlock_page(rpage); 
	// }
	int num_to_skip=0;
	for (int i = 0; i < dic->cluster_size; i+=num_to_skip) {
		struct folio *folio;
		num_to_skip=1;
		if (!dic->rpages[i])
			continue;
		folio = page_folio(dic->rpages[i]);	
		while ((i + num_to_skip) < dic->cluster_size && dic->rpages[i + num_to_skip] &&
		       page_folio(dic->rpages[i + num_to_skip]) == folio) {
			num_to_skip++;
		}
		struct f2fs_iomap_folio_state *fifs=folio->private;
		if(failed)
		{
			if(folio_order(folio)>0&&fifs)
			{
				f2fs_ifs_clear_range_uptodate(folio,fifs,0,num_to_skip<<PAGE_SHIFT);
				folio_clear_uptodate(folio);
				folio_unlock(folio);
			}
			else
			{
				folio_clear_uptodate(folio);
				folio_unlock(folio);
			}
		}
		else
		{	
			#ifdef CONFIG_F2FS_DEBUG_PRINT
			f2fs_err(F2FS_I_SB(dic->inode),"from %s:",__func__);
			#endif
			f2fs_iomap_finish_folio_read(folio,0,num_to_skip<<PAGE_SHIFT,0);
		}
	}
	/*
	 * Release the reference to the decompress_io_ctx that was being held
	 * for I/O completion.
	 */
	f2fs_put_dic(dic, in_task);
}
```
这些代码不但要详细地讲到我进行的large folios相关改动,还得讲到最终取消掉read_bytes_pending的地方。你可能没看到。但是它在这里:
```C
void f2fs_iomap_finish_folio_read(struct folio *folio, size_t off,size_t len, int error)
{
	// #ifdef CONFIG_F2FS_DEBUG_PRINT
	// if(folio_test_locked(folio))
	// {
	// 	f2fs_err(F2FS_F_SB(folio),"%s yes folio index %d order %d host ino %d is locked",__func__,folio->index,folio_order(folio),folio_inode(folio)->i_ino);
	// }
	// else
	// {
	// 	f2fs_err(F2FS_F_SB(folio),"%s no!!! folio index %d order %d host ino %d is not locked",__func__,folio->index,folio_order(folio),folio_inode(folio)->i_ino);
	// }
	// #endif
	struct f2fs_iomap_folio_state *fifs =folio->private;
		bool uptodate = !error;
		bool finished = true;
		if(folio_order(folio)>0&&fifs)
		{
			unsigned long flags;
			spin_lock_irqsave(&fifs->state_lock, flags);
			fifs->read_bytes_pending-=len;
			#ifdef CONFIG_F2FS_DEBUG_PRINT
			f2fs_err(F2FS_F_SB(folio),"%s after rbp sub:%d",__func__);
			FUNC(print_rbp,folio);
			#endif
			if(!error)
			{	
				uptodate=ifs_set_range_uptodate(folio, (struct iomap_folio_state*)fifs,off,len);
			}
			finished = (fifs->read_bytes_pending == 0||fifs->read_bytes_pending==F2FS_IFS_MAGIC);
			spin_unlock_irqrestore(&fifs->state_lock, flags);
			if(finished)
			{
				folio_end_read(folio, uptodate);
			}
		}
		else
		{
			folio_end_read(folio, !error);
		}
}
```
是我抽象出来的一个共通的函数。
请十分详细地缕清这些函数的逻辑并详细地阐释他们。为了给你一个完全的上下文,我给你一个全部的函数上下文定义:
```C
__attribute__((optimize("O0"))) 
static int f2fs_compress_readpage_iter(struct iomap_iter *iter,struct f2fs_readpage_ctx *ctx)
{
	loff_t pos = iter->pos;
	loff_t length = iomap_length(iter);
	struct folio *folio = ctx->cur_folio;
	struct inode* inode=folio->mapping->host;
	struct f2fs_iomap_folio_state *fifs;
	size_t poff, plen;
	int ret;
#ifdef CONFIG_F2FS_FS_COMPRESSION
	bool is_compressed =
		f2fs_is_compressed_cluster(inode, pos >> PAGE_SHIFT);
#endif
	/* zero post-eof blocks as the page may be mapped */
	fifs = f2fs_ifs_alloc(folio, iter->flags,false);
	iomap_adjust_read_range(iter->inode, folio, &pos, length, &poff, &plen);
	if (plen == 0)
		goto done;
	if(!is_compressed)
	{
		if (iomap_block_needs_zeroing(iter, pos)) {
			folio_zero_range(folio, poff, plen);
			iomap_set_range_uptodate(folio, poff, plen);
			goto done;
		}
	}
	if (is_compressed) {
		ret = f2fs_do_read_multi_folios(ctx,pos,plen);
	} else {
		ret = f2fs_do_read_single_folio_iomap(iter, ctx, pos, plen, poff);
	}
	if(ret)
		return ret;
done:
	length = pos - iter->pos + plen;
	return iomap_iter_advance(iter, &length);
}
__attribute__((optimize("O0")))
static loff_t f2fs_compress_readahead_iter(struct iomap_iter *iter,struct f2fs_readpage_ctx *ctx)
{
	int ret = 0;
	while (iomap_length(iter)) {
		if (ctx->cur_folio &&offset_in_folio(ctx->cur_folio, iter->pos) == 0) {
			if (!ctx->cur_folio_in_bio &&
			    !ctx->cur_folio_in_compress_ctx)
				folio_unlock(ctx->cur_folio);
			ctx->cur_folio = NULL;
		}
		if (!ctx->cur_folio) {
			ctx->cur_folio = readahead_folio(ctx->rac);
			ctx->cur_folio_in_bio = false;
			ctx->cur_folio_in_compress_ctx=false;
		}
		ret = f2fs_compress_readpage_iter(iter, ctx);
		if (ret)
			return ret;
	}
	return 0;
}
__attribute__((optimize("O0")))
int f2fs_compress_iomap_readahead(struct inode *inode,
				  struct readahead_control *rac)
{
	struct f2fs_readpage_ctx ctx;
	int ret = 0;
	f2fs_init_readpage_ctx(&ctx, rac);
	struct iomap_iter iter = {
		.inode = inode,
		.pos = readahead_pos(rac),
		.len = readahead_length(rac),
	};
	while ((ret = iomap_iter(
			&iter, &f2fs_buffered_read_compress_iomap_ops)) > 0) {
		iter.status = f2fs_compress_readahead_iter(&iter, &ctx);
	}
	if (ctx.bio) {
		f2fs_submit_read_bio(
			F2FS_I_SB(inode), ctx.bio,
			 DATA);
		ctx.bio = NULL;
	}
	if (ctx.cur_folio && !ctx.cur_folio_in_bio &&
	    !ctx.cur_folio_in_compress_ctx) {
		folio_unlock(ctx.cur_folio);
	}
	// f2fs_destroy_readpage_ctx(&ctx);
	#ifdef CONFIG_F2FS_DEBUG_PRINT
	ssleep(1);
	FUNC(f2fs_check_inode_folios_writeback, inode);
	#endif
	return ret;	
}
```
我包裹在debug print的代码你知道是调试代码就行了。
