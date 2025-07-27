## 功能背景（就不和下面的重大问题分析撞车了)
我们已经决定将整个buffered write路径迁移往iomap了,因为经过我们前面的分析我们知道这是最无包袱的支持large folios的方式。而和buffered read和page writeback
不同的一点是,iomap_buffered_write除了提供iomap_begin和iomap_end,还提供了我们可以自己定制化获取folio逻辑和释放folio逻辑的__iomap_get_folio和__iomap_put_folio。通过
分析,我发现用iomap buffered write实现压缩文件的buffered write是可能的并且函数栈的调用路径可以做到更少的调用栈开销。
## 重大问题剖析
初赛时候我们的设计方案
是我们将改造成支持large folios的prepare write begin逻辑 
也即原先的buffered write的逻辑 给拿过来填入到iomap_begin之中。然后__iomap_get_folio之中我们和原代码逻辑一样,从压缩文件上下文中根据我们的索引取出里面的page转化为folio
结构体。但是因为iomap_get_folio的策略和我们在iomap_folio_state中存储的length是只被限制在一个簇之中 这里我们要贴出iomap_get_folio中传入阶数hint的地方了。
```C
iomap_get_folio
```
所以我发现我们几乎是严重限制了large folios针对buffered write的性能 因为我们大大地限制了其能分配的folio的上限

## 方案设计
```C
static int f2fs_buffered_write_iomap_begin(struct inode *inode, loff_t pos, loff_t length,
				unsigned flags, struct iomap *iomap,
				struct iomap *srcmap)
{
	struct f2fs_sb_info *sbi = F2FS_I_SB(inode);
	struct f2fs_map_blocks map = {};
	struct folio *ifolio = NULL;
	int err = 0;
	iomap->offset = pos; 
	iomap->bdev = sbi->sb->s_bdev; // Default block device
	iomap->dax_dev = NULL;
	iomap->private = NULL;
	iomap->folio_ops = &f2fs_iomap_folio_ops;
	iomap->flags = 0; 
#ifdef CONFIG_F2FS_FS_COMPRESSION
	pgoff_t index = pos >> PAGE_SHIFT;
	struct compress_ctx cc = {
			.inode = inode,
			.log_cluster_size = F2FS_I(inode)->i_log_cluster_size,
			.cluster_size = F2FS_I(inode)->i_cluster_size,
			.cluster_idx = cluster_i_idx(inode,index),
		};
	if (pos<i_size_read(inode)&&f2fs_compressed_file(inode)&&f2fs_is_compressed_cluster(inode,index)) {
		length=min_t(loff_t,length,i_size_read(inode)-pos);
		struct iomap_iter* iter=container_of(iomap,struct iomap_iter,iomap);
		struct bio*bio=NULL;
		loff_t new_pos = pos;
		size_t poff;
		size_t plen;
		pgoff_t start_idx = start_idx_of_cluster(&cc);//Aligned to cluster's start page index
		pgoff_t end_idx = ((pos+length)>>PAGE_SHIFT)-1;
		end_idx = end_idx_of_cluster(&cc,end_idx);//Aligned end idx to the last cluster's end
		loff_t aligned_len=(end_idx-start_idx+1)<<PAGE_SHIFT;
		#ifdef CONFIG_F2FS_DEBUG_PRINT
		ssleep(1);
		f2fs_err(F2FS_I_SB(inode),"current state:pos%d,end_idx%d",pos,end_idx);
		FUNC(f2fs_check_inode_folios_writeback, inode);
		#endif // DEBUG
		struct folio*folio=iomap_get_folio(iter,start_idx<<PAGE_SHIFT,aligned_len);
		if (folio_test_uptodate(folio))
		{
			iomap->length+=min_t(unsigned int,folio_nr_pages(folio)<<PAGE_SHIFT,length);
			goto setup_iomap;
		}
		struct f2fs_iomap_folio_state* fifs=f2fs_ifs_alloc(folio, iter->flags,false);
		if(folio_order(folio)>0&&fifs)
		{
			unsigned long flags;
			spin_lock_irqsave(&fifs->state_lock, flags);
			fifs->read_bytes_pending+=1;//Add a bias for read bytes_pending so folio_end_read will never be called
			spin_unlock_irqrestore(&fifs->state_lock, flags);
		}
		iomap_adjust_read_range(inode,folio,&new_pos,aligned_len,&poff, &plen);
		end_idx =min(end_idx,folio->index+folio_nr_pages(folio));
		for (int i=cluster_idx(&cc,new_pos>>PAGE_SHIFT)<<cc.log_cluster_size;;)
		{
    		if (cc.nr_cpages==NULL)
			{
			err = f2fs_init_compress_ctx(&cc);
			if (err < 0)
			goto out_unlock;
			}
    		loff_t add_len=min(cc.cluster_size,end_idx-i+1)<<PAGE_SHIFT;
    		do_read_multi_folios(&cc,folio,i<<PAGE_SHIFT,add_len,&bio,0,true);
			iomap->length+=add_len;
    		i+=cc.cluster_size;
    		if(i>=end_idx||!f2fs_is_compressed_cluster(cc.inode, i))
        		break;
		}
		bool last_cluster_is_partial=false;
		if(f2fs_cluster_is_partial_full(&cc))
		{
			last_cluster_is_partial=true;
			for (pgoff_t idx = end_idx+1; idx < end_idx_of_cluster(&cc,end_idx); idx++) {
				struct folio *extra_folio = iomap_get_folio(iter, (loff_t)idx << PAGE_SHIFT, PAGE_SIZE);
				if (IS_ERR(extra_folio)) { 
					f2fs_folio_put(extra_folio,true);
				 }
				do_read_multi_folios(&cc, extra_folio, (loff_t)idx << PAGE_SHIFT, PAGE_SIZE, &bio, NULL, true);
			}
		}
		if (bio)
			f2fs_submit_read_bio(sbi, bio, DATA);
		if (folio_order(folio)>0&&fifs) {
			/*We use read_bytes_pending to wait for the whole folio to be read
			We didn't use submit_bio_wait because folio can cross multi bios
			*/
			
			for(;;)
			{
				bool done;
				unsigned long flags;
				spin_lock_irqsave(&fifs->state_lock, flags);
				done = (fifs->read_bytes_pending == F2FS_IFS_MAGIC+1);
				spin_unlock_irqrestore(&fifs->state_lock, flags);
				if (done)
					break;
				f2fs_io_schedule_timeout(DEFAULT_IO_TIMEOUT);
			}
		}	
			if (last_cluster_is_partial) {
				// 如果最后一个 cluster 是部分填充的，需要额外检查整个 cluster 的状态
				pgoff_t cluster_start = cluster_idx(&cc, index)<<cc.log_cluster_size;
				err = f2fs_wait_cluster_uptodate(inode, end_idx+1, cc.cluster_size);
				if (err) goto out_clean_bias;
			}
		if(folio_order(folio)>0&&fifs)
		{
			spin_lock_irqsave(&fifs->state_lock, flags);
			fifs->read_bytes_pending-=1;//Add a bias for read bytes_pending so folio_end_read will never be called
			spin_unlock_irqrestore(&fifs->state_lock, flags);
		}
		setup_iomap:
		iomap->private = folio; 
		iomap->type = IOMAP_MAPPED; 
		iomap->addr = IOMAP_NULL_ADDR; 
		iomap->offset = pos;
		iomap->bdev = sbi->sb->s_bdev;
		iomap->dax_dev = NULL;
		iomap->flags = 0;
		return 0;
		out_unlock:
		folio_unlock(folio);
		folio_put(folio);
		out_clean_bias:
		if (fifs)
		{
			spin_lock_irqsave(&fifs->state_lock, flags);
			fifs->read_bytes_pending-=1;
			spin_unlock_irqrestore(&fifs->state_lock, flags);
		}
		return err;
		}
```
我们切换了视角 不再是以cluster的视角出发 一点一点用folio将cluster给填满 转而切换为从一个大folio的视角出发 不断地用cluster去填充它特别强调这段是我们自己的原创算法
然后我们将整个处理好的folio就保持着上锁的状态 然后将其放到iomap->private之中 送入iomap的主要处理逻辑。注意在我们iomap_begin中这样处理的逻辑 最终总是会让我们拿到的这个folio全为uptodate
这里面还有个非常重要的点。f2fs内提交io的代码全部都是异步的。但是我们这个地方和所有的 iomap buffered write一样是必须等到整个folio全部给读完 直到全部uptodate 我们才能对其进行写入，
不然就会出现还没等folio内容全部从磁盘读上来,就被buffered write写入造成的严重数据不一问题。而我们面临的问题是f2fs原生的代码并没直接提供给我们这样进行同步等待的设施,而内核虽然具有submit_bio_wait,但是它只能做到等待一个bio级别部分被完成,没法做到等待folio的全部部分。于是,这个时候我想到有一种很好的利用`read_bytes_pending`的主意,首先我们先加上一个bias,这样我们就能确保所有的bio回调不会被调用(因为始终有这个bias阻止它们调用)，然后我们显式地在iomap_begin中轮询直到read_bytes_pending的值刚好是f2fs_ifs_magic+1.此时folio必然是全uptodate状态。同时为了避免出现有一个压缩簇里只有部分folio处于uptodate这种尴尬的情况,（尤其是这个folio本身就不够覆盖这个簇的情况下)我还显式地填满了这个folio最后一个所在的簇。
