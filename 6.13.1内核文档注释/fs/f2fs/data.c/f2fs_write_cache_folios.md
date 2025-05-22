对比着现在内核最新版本实现的`iomap_writepages`以及`write_cache_pages`,我们也尝试使用基于`writeback_iter`的方式,重写f2fs中的`f2fs_write_cache_pages`方法吧。
```C
static int f2fs_write_cache_folios(struct address_space *mapping,
				  struct writeback_control *wbc,
				  enum iostat_type io_type)
{
	struct iomap_writepage_ctx wpc={
		.io_type = io_type,
	}
#ifdef CONFIG_F2FS_FS_COMPRESSION
	struct inode *inode = mapping->host;
	struct compress_ctx cc = {
		.inode = inode,
		.log_cluster_size = F2FS_I(inode)->i_log_cluster_size,
		.cluster_size = F2FS_I(inode)->i_cluster_size,
		.cluster_idx = NULL_CLUSTER,
		.rfolios = NULL,
		.nr_rpages = 0,
        .last_idx=0,
        .nr_remains=F2FS_I(inode)->i_cluster_size,
		.cfolios = NULL,
		.valid_nr_cpages = 0,
		.rbuf = NULL,
		.cbuf = NULL,
		.rlen = PAGE_SIZE * F2FS_I(inode)->i_cluster_size,
		.private = NULL,
	};
#endif
    struct folio* folio=NULL;
    int error;
    if (get_dirty_pages(mapping->host) <=
	    SM_I(F2FS_M_SB(mapping))->min_hot_blocks)
		set_inode_flag(mapping->host, FI_HOT_DATA);
	/*如果脏页数量小于等于 min_hot_blocks 阈值，
		则将 inode 标记为 FI_HOT_DATA (热数据)，否则清除该标记 (冷数据)。*/
	else
		clear_inode_flag(mapping->host, FI_HOT_DATA);
    /*我们来说一下writeback_iter都干了什么事情啊。
    首先index根据是否是回环写这个判断已经写好了并且我们也不用计算结束索引,也不用手动进行folio_batch_init。然后if (wbc->sync_mode 
    == WB_SYNC_ALL || wbc->tagged_writepages)
		tag_pages_for_writeback(mapping, index, end);这个逻辑是一模一样的,少了else
		tag = PAGECACHE_TAG_DIRTY这个分支。
        然后 关于等待folio写回以及其是为脏的这些同步问题,writeback_iter内部已经帮我们解决了*/
    while(folio=writeback_iter(mapping,wbc,folio,&error))
    {
        /*这里开始就和f2fs_write_cache_pages原先的逻辑有很大的区别了
        在考虑和truncate并发的时候,原f2fs_write_cache_pages函数是将从
        filemap_get_folios_tag拿到的folio拆散成page,然后再对它们一一上锁。
        并且检查此时mapping还在不在了。但是writeback_iter这里,我们一次只能看到一整个folio,所有的并发了,等待回写操作了,全是只基于这个folio单元的。因此我们上来就直接对这单个folio做并发上的检查*/
    struct f2fs_iomap_folio_state *ifs = folio->private;
	struct inode *inode = folio->mapping->host;
	u64 pos = folio_pos(folio);
	u64 end_pos = pos + folio_size(folio);
	u64 end_aligned = 0;
	unsigned count = 0;
	int error = 0;
	u32 rlen;

	WARN_ON_ONCE(!folio_test_locked(folio));
	WARN_ON_ONCE(folio_test_dirty(folio));
	WARN_ON_ONCE(folio_test_writeback(folio));
	if (!iomap_writepage_handle_eof(folio, inode, &end_pos)) {
		folio_unlock(folio);
		return 0;/*iomap_writepage_map这里return 0实际上相当于循环continue一次*/
	}
    folio_start_writeback(folio);
    end_aligned = round_up(end_pos, i_blocksize(inode));
    }
    /*我们预期的f2fs_iomap_find_dirty_range函数无论是寻找脏的cluster 还是寻找脏的page区间,每次只会返回一段连续的脏区间。*/
	while ((rlen = iomap_find_dirty_range(folio, &pos, end_aligned))) {
		error = f2fs_iomap_writepage_map_blocks(wpc,wbc, folio, inode,
				pos, end_pos, rlen, &count);
		/*我感觉可以把io_type给存到writepage_ctx之中*/
		/*begin f2fs_iomap_writepage_map_blocks*/
		/*注意我们已经处理了eof*/
		/*第一步 计算实际能映射得到的区间并且我打算在这一步就将fwpc中的iomap_end中的bio分配好。*/
		error = wpc->ops->map_blocks(wpc, inode, pos, dirty_len);
		/*begin wpc->ops->map_blocks*/
		struct f2fs_map_blocks map;
		block_t start_blk=F2FS_BYTES_TO_BLK(pos);
		block_t len_blks=F2FS_BYTES_TO_BLK(offset + length - 1) - start_blk + 1;
		err=f2fs_map_blocks_iomap(start_blk,len_blks,&map);
		
		if (error)
			break;
		pos += rlen;
	}
	
}
```