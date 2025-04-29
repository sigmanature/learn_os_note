**相关函数**
* [f2fs_write_begin](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/data.c/f2fs_write_begin.md)
进行buffered write的时候,为压缩文件进行准备的函数。初始化上下文之后直接调用`prepare_compress_overwrite`
```C
int f2fs_prepare_compress_overwrite(struct inode *inode,
		struct page **pagep, pgoff_t index, void **fsdata)
{
	struct compress_ctx cc = {
		.inode = inode,
		.log_cluster_size = F2FS_I(inode)->i_log_cluster_size,
		.cluster_size = F2FS_I(inode)->i_cluster_size,
		.cluster_idx = index >> F2FS_I(inode)->i_log_cluster_size,
		.rpages = NULL,
		.nr_rpages = 0,
	};

	return prepare_compress_overwrite(&cc, pagep, index, fsdata);
}
```
```C
static int prepare_compress_overwrite(struct compress_ctx *cc,
		struct page **pagep, pgoff_t index, void **fsdata)
{
	struct f2fs_sb_info *sbi = F2FS_I_SB(cc->inode);
	struct address_space *mapping = cc->inode->i_mapping;
	struct folio *folio;
	sector_t last_block_in_bio;
	fgf_t fgp_flag = FGP_LOCK | FGP_WRITE | FGP_CREAT;
	pgoff_t start_idx = start_idx_of_cluster(cc);
	int i, ret;

retry:
	ret = f2fs_is_compressed_cluster(cc->inode, start_idx);
	if (ret <= 0)
		return ret;

	ret = f2fs_init_compress_ctx(cc);
	if (ret)
		return ret;

	/* keep folio reference to avoid page reclaim */
	for (i = 0; i < cc->cluster_size; i++) {
		folio = f2fs_filemap_get_folio(mapping, start_idx + i,
				fgp_flag, GFP_NOFS);
		if (IS_ERR(folio)) {
			ret = PTR_ERR(folio);
			goto unlock_pages;
		}

		if (folio_test_uptodate(folio))
			f2fs_folio_put(folio, true);
		else
			f2fs_compress_ctx_add_page(cc, folio);
	}

	if (!f2fs_cluster_is_empty(cc)) {
		struct bio *bio = NULL;

		ret = f2fs_read_multi_pages(cc, &bio, cc->cluster_size,
					&last_block_in_bio, NULL, true);
		f2fs_put_rpages(cc);
		f2fs_destroy_compress_ctx(cc, true);
		if (ret)
			goto out;
		if (bio)
			f2fs_submit_read_bio(sbi, bio, DATA);

		ret = f2fs_init_compress_ctx(cc);
		if (ret)
			goto out;
	}

	for (i = 0; i < cc->cluster_size; i++) {
		f2fs_bug_on(sbi, cc->rpages[i]);

		folio = filemap_lock_folio(mapping, start_idx + i);
		if (IS_ERR(folio)) {
			/* folio could be truncated */
			goto release_and_retry;
		}

		f2fs_folio_wait_writeback(folio, DATA, true, true);
		f2fs_compress_ctx_add_page(cc, folio);

		if (!folio_test_uptodate(folio)) {
			f2fs_handle_page_eio(sbi, folio, DATA);
release_and_retry:
			f2fs_put_rpages(cc);
			f2fs_unlock_rpages(cc, i + 1);
			f2fs_destroy_compress_ctx(cc, true);
			goto retry;
		}
	}

	if (likely(!ret)) {/*如果说ret为0表示正常返回了*/
		*fsdata = cc->rpages;/*整个rpages数组被返回回去了?*/
		*pagep = cc->rpages[offset_in_cluster(cc, index)];/*这就是把刚才已经被读上来的那个page给返回了吧*/
		return cc->cluster_size;
	}

unlock_pages:
	f2fs_put_rpages(cc);
	f2fs_unlock_rpages(cc, i);
	f2fs_destroy_compress_ctx(cc, true);
out:            
	return ret;
}
```
这个函数在`f2fs_write_begin`中的调用上下文中是这样的:
```C
static int f2fs_write_begin(struct file *file, struct address_space *mapping,
		loff_t pos, unsigned len, struct folio **foliop, void **fsdata)
#ifdef CONFIG_F2FS_FS_COMPRESSION
	if (f2fs_compressed_file(inode)) {
		int ret;
		struct page *page;

		*fsdata = NULL;

		if (len == PAGE_SIZE && !(f2fs_is_atomic_file(inode)))
			goto repeat;

		ret = f2fs_prepare_compress_overwrite(inode, &page,
							index, fsdata);
		if (ret < 0) {
			err = ret;
			goto fail;
		} else if (ret) {/*ret为0可能是表示不是压缩簇。这个时候两条分支都不会走
                        很大概率是切换到了对待普通文件的分支上去了。*/
			*foliop = page_folio(page);/*ret为正的场合表示是压缩簇 此时这个page/folio的内容已经被读上来了
            所以的话直接返回这个page就行了*/
			return 0;
		}
	}
#endif
``` 
从`f2fs_prepare_compress_overwrite(inode, &page,
							index, fsdata)`中返回回来的fsdata指针,也就是整个rpage指针,在之后整个buffered write中发挥的作用只有在f2fs_write_end中被使用。
```C
static int f2fs_write_end(struct file *file,
			struct address_space *mapping,
			loff_t pos, unsigned len, unsigned copied,
			struct folio *folio, void *fsdata)
{
	struct inode *inode = folio->mapping->host;

	trace_f2fs_write_end(inode, pos, len, copied);

	/*
	 * This should be come from len == PAGE_SIZE, and we expect copied
	 * should be PAGE_SIZE. Otherwise, we treat it with zero copied and
	 * let generic_perform_write() try to copy data again through copied=0.
	 */
	if (!folio_test_uptodate(folio)) {
		if (unlikely(copied != len))
			copied = 0;
		else
			folio_mark_uptodate(folio);
	}

#ifdef CONFIG_F2FS_FS_COMPRESSION
	/* overwrite compressed file */
	if (f2fs_compressed_file(inode) && fsdata) {
		f2fs_compress_write_end(inode, fsdata, folio->index, copied);/*核心逻辑在这吧*/
		f2fs_update_time(F2FS_I_SB(inode), REQ_TIME);

		if (pos + copied > i_size_read(inode) &&
				!f2fs_verity_in_progress(inode))
			f2fs_i_size_write(inode, pos + copied);
		return copied;
	}
#endif

	if (!copied)
		goto unlock_out;

	folio_mark_dirty(folio);

	if (f2fs_is_atomic_file(inode))
		set_page_private_atomic(folio_page(folio, 0));

	if (pos + copied > i_size_read(inode) &&
	    !f2fs_verity_in_progress(inode)) {
		f2fs_i_size_write(inode, pos + copied);
		if (f2fs_is_atomic_file(inode))
			f2fs_i_size_write(F2FS_I(inode)->cow_inode,
					pos + copied);
	}
unlock_out:
	folio_unlock(folio);
	folio_put(folio);
	f2fs_update_time(F2FS_I_SB(inode), REQ_TIME);
	return copied;
}
```