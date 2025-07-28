## 功能背景
保持整体内容框架 但是可以描述得更加详细一点。
## 重大问题分析
保持整体内容框架不变 但可以描述得更加详细一点。
## 方案设计
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
