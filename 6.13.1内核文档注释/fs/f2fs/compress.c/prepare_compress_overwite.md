```C
int f2fs_prepare_compress_overwrite(struct inode *inode,
		struct page **pagep, pgoff_t index, void **fsdata)
{
	struct compress_ctx cc = {
		.inode = inode,
		.log_cluster_size = F2FS_I(inode)->i_log_cluster_size,
		.cluster_size = F2FS_I(inode)->i_cluster_size,
		.cluster_idx = index >> F2FS_I(inode)->i_log_cluster_size,
		.rpages = NULL,/*栈上分配的上下文结构体还是初始的时候page cache中的pages数组是空的*/
		.nr_rpages = 0,
	};

	return prepare_compress_overwrite(&cc, pagep, index, fsdata);
}
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

	if (likely(!ret)) {
		*fsdata = cc->rpages;
		*pagep = cc->rpages[offset_in_cluster(cc, index)];
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