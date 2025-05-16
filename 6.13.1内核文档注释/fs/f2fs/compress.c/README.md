这个板块我们仔细地梳理一下f2fs中的压缩文件的那些page都是从哪里分配的。
```C
int f2fs_read_multi_pages(struct compress_ctx *cc, struct bio **bio_ret,
				unsigned nr_pages, sector_t *last_block_in_bio,
				struct readahead_control *rac, bool for_write)
{
    /* get rid of pages beyond EOF */
	for (i = 0; i < cc->cluster_size; i++) {
		struct page *page = cc->rpages[i];
        /*这些page全部来自page_cache,是正儿八经的文件页面*/
        /*...*/
    }
    f2fs_alloc_dic(&cc)
    /*begin f2fs_alloc_dic*/
    struct decompress_io_ctx *dic;
    dic->rpages = page_array_alloc(cc->inode, cc->cluster_size);
    for (i = 0; i < dic->cluster_size; i++)
		dic->rpages[i] = cc->rpages[i];
    /*解压上下文自己的rpages数组复制自cc中正儿八经的页面的page*/
    /*end f2fs_alloc_dic*/
    dic->cpages = page_array_alloc(dic->inode, dic->nr_cpages);
	if (!dic->cpages) {
		ret = -ENOMEM;
		goto out_free;
	}
	for (i = 0; i < dic->nr_cpages; i++) {
		struct page *page;

		page = f2fs_compress_alloc_page();
		f2fs_set_compressed_page(page, cc->inode,
					start_idx + i + 1, dic);
        /*解压上下文的cpage则全部来自于内存池,并且它们的private全部
        被设置为dic。争议在这。这些page/folio 需要分配ifs吗?
        还是说只要存private字段就够了?*/
		dic->cpages[i] = page;
	}
}
```