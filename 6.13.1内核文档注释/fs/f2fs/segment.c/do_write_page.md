```C
static void do_write_page(struct f2fs_summary *sum, struct f2fs_io_info *fio)
{
	enum log_type type = __get_segment_type(fio);
	int seg_type = log_type_to_seg_type(type);
	bool keep_order = (f2fs_lfs_mode(fio->sbi) &&
				seg_type == CURSEG_COLD_DATA);

	if (keep_order)
		f2fs_down_read(&fio->sbi->io_order_lock);

	if (f2fs_allocate_data_block(fio->sbi, fio->page, fio->old_blkaddr,
			&fio->new_blkaddr, sum, type, fio)) {
		if (fscrypt_inode_uses_fs_layer_crypto(fio->page->mapping->host))
			fscrypt_finalize_bounce_page(&fio->encrypted_page);
		end_page_writeback(fio->page);
		if (f2fs_in_warm_node_list(fio->sbi, fio->page))
			f2fs_del_fsync_node_entry(fio->sbi, fio->page);
		goto out;
	}
	if (GET_SEGNO(fio->sbi, fio->old_blkaddr) != NULL_SEGNO)
		f2fs_invalidate_internal_cache(fio->sbi, fio->old_blkaddr, 1);

	/* writeout dirty page into bdev */
	f2fs_submit_page_write(fio);

	f2fs_update_device_state(fio->sbi, fio->ino, fio->new_blkaddr, 1);
out:
	if (keep_order)
		f2fs_up_read(&fio->sbi->io_order_lock);
}
```