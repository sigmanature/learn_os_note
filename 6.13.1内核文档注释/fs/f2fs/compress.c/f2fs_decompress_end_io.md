```C
/*
 * This is called when a compressed cluster has been decompressed
 * (or failed to be read and/or decompressed).
 */
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
	for (i = 0; i < dic->cluster_size; i++) {
		struct page *rpage = dic->rpages[i];

		if (!rpage)
			continue;

		if (failed) 
			ClearPageUptodate(rpage);
		else
			SetPageUptodate(rpage);
		unlock_page(rpage);
	}

	/*
	 * Release the reference to the decompress_io_ctx that was being held
	 * for I/O completion.
	 */
	f2fs_put_dic(dic, in_task);
}
```
我们来仔细分析一下f2fs_compress_end_io的调用上下文。
1. 在[f2fs_read_multi_pages](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/data.c/f2fs_read_multi_pages)之中
```C
int f2fs_read_multi_pages(struct compress_ctx *cc, struct bio **bio_ret,
				unsigned nr_pages, sector_t *last_block_in_bio,
				struct readahead_control *rac, bool for_write)
{
    for (i = 0; i < cc->nr_cpages; i++) {
		if (f2fs_load_compressed_page(sbi, folio_page(folio, 0),blkaddr)) {
            /*atomic_dec_and_test只有在被减的变量值归0
            才会返回true。此时进行正常的解压缩逻辑。*/
			if (atomic_dec_and_test(&dic->remaining_pages)) {
				f2fs_decompress_cluster(dic, true);
                /*f2fs_decompress_end_io会调用f2fs_decompress_end_io。也就是在一个簇被正常解压完了 才会把所有的page都标记成uptodate*/
				break;
			}
			continue;
		}

		if (bio && (!page_is_mergeable(sbi, bio,
					*last_block_in_bio, blkaddr) ||
		    !f2fs_crypt_mergeable_bio(bio, inode, folio->index, NULL))) {
        submit_and_realloc:
			f2fs_submit_read_bio(sbi, bio, DATA);
			bio = NULL;
		}

		if (!bio) {
			bio = f2fs_grab_read_bio(inode, blkaddr, nr_pages,
					f2fs_ra_op_flags(rac),
					folio->index, for_write);
			if (IS_ERR(bio)) {
				ret = PTR_ERR(bio);
                /*出现失败的时候啊,f2fs_decompress_end_io会把cluster中的所有rpage全部清除uptodate标记*/
				f2fs_decompress_end_io(dic, ret, true);
				f2fs_put_dnode(&dn);
				*bio_ret = NULL;
				return ret;
			}
		}
}
```
