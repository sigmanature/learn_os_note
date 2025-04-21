**相关函数**
* [f2fs_do_write_page](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/data.c/f2fs_do_write_page.md)
 ```c
/**
 * f2fs_write_single_data_page - Write a single data page to disk.
 * @folio:		The folio containing the page to write.
 * @submitted:		Output parameter, set to 1 if bio was submitted, 0 otherwise.
 * @bio:		Input/Output. Pointer to a bio structure for potential merging.
 * @last_block:		Input/Output. Pointer to the last block sector written for merging.
 * @wbc:		Writeback control structure.
 * @io_type:		IO type for statistics (e.g., sync/async).
 * @compr_blocks:	Number of compressed blocks if writing compressed data.
 * @allow_balance:	Boolean indicating if f2fs_balance_fs() can be called.
 *
 * This function handles the process of writing a single data page (within a folio)
 * from the page cache to the F2FS filesystem. It deals with finding block
 * addresses, handling inline data, compression, EOF conditions, merging BIOs,
 * and interacting with the checkpoint mechanism.
 *
 * Context: This function is typically called by f2fs_write_cache_pages() via
 *          the address_space_operations ->writepage callback.
 *
 * Return: 0 on success (page handled), AOP_WRITEPAGE_ACTIVATE if page should
 *         be redirtied and kept active, or a negative error code on failure.
 */
int f2fs_write_single_data_page(struct folio *folio, int *submitted,
				struct bio **bio,
				sector_t *last_block,
				struct writeback_control *wbc,
				enum iostat_type io_type,
				int compr_blocks,
				bool allow_balance)
{
	// Get the inode associated with this page/folio's mapping.
	struct inode *inode = folio->mapping->host;
	// Extract the struct page pointer from the folio (assuming single page folio for now).
	struct page *page = folio_page(folio, 0);
	// Get the F2FS superblock info from the inode.
	struct f2fs_sb_info *sbi = F2FS_I_SB(inode);
	// Read the current size of the inode (file size).
	loff_t i_size = i_size_read(inode);
	// Calculate the page index corresponding to the end of the file.
	const pgoff_t end_index = ((unsigned long long)i_size)
							>> PAGE_SHIFT;
	// Calculate the byte offset corresponding to the end of the *current* folio.
	loff_t psize = (loff_t)(folio->index + 1) << PAGE_SHIFT;
	// Initialize the offset within the page (used for partial pages at EOF).
	unsigned offset = 0;
	// Flag to indicate if f2fs_balance_fs should be called later.
	bool need_balance_fs = false;
	// Check if this is a quota inode (has special handling, less aggressive writeback).
	bool quota_inode = IS_NOQUOTA(inode);
	// Initialize error code to 0 (success).
	int err = 0;
	// Structure to hold all information needed for the lower-level write operation.
	struct f2fs_io_info fio = {
		.sbi = sbi,				// Filesystem superblock info
		.ino = inode->i_ino,			// Inode number
		.type = DATA,				// Type of page being written (Data)
		.op = REQ_OP_WRITE,			// Operation type (Write)
		.op_flags = wbc_to_write_flags(wbc),	// Operation flags (e.g., REQ_SYNC) from wbc
		.old_blkaddr = NULL_ADDR,		// Previous block address (for overwrite, initially none)
		.page = page,				// The page structure itself
		.encrypted_page = NULL,			// Placeholder for encrypted page
		.submitted = 0,				// Flag: Has a BIO been submitted yet by f2fs_do_write_data_page?
		.compr_blocks = compr_blocks,		// Number of compressed blocks (0 if not compressed)
		// Locking strategy: If compressed, assume lock handled elsewhere (LOCK_DONE).
		// Otherwise, try without lock first (LOCK_RETRY), acquire later if needed.
		.need_lock = compr_blocks ? LOCK_DONE : LOCK_RETRY,
		// Flag indicating if metadata GC is needed for this inode (typically for meta inodes).
		.meta_gc = f2fs_meta_inode_gc_required(inode) ? 1 : 0,
		.io_type = io_type,			// IO type for stats
		.io_wbc = wbc,				// Pass the writeback control structure
		.bio = bio,				// Pointer to the current BIO pointer (for merging)
		.last_block = last_block,		// Pointer to the last block sector (for merging)
	};

	// Tracepoint for f2fs page write operations.
	trace_f2fs_writepage(folio, DATA);

	// Check if the filesystem has encountered a checkpoint error.
	if (unlikely(f2fs_cp_error(sbi))) {
		// Mark the address space mapping as having an I/O error.
		mapping_set_error(folio->mapping, -EIO);
		/*
		 * During shutdown (SBI_IS_CLOSE), we might still try to write.
		 * For directories, don't discard dirty pages if not closing,
		 * to preserve the latest directory structure info.
		 */
		if (S_ISDIR(inode->i_mode) &&
				!is_sbi_flag_set(sbi, SBI_IS_CLOSE))
			goto redirty_out; // Redirty the page and try again later.

		// If mount option is remount-ro on error, keep data pages dirty.
		if (F2FS_OPTION(sbi).errors == MOUNT_ERRORS_READONLY)
			goto redirty_out; // Redirty the page.
		// Otherwise (e.g., errors=panic), just bail out without writing or redirtying.
		goto out;
	}

	// Check if Power-On Recovery (POR) is in progress. Writes are generally blocked during POR.
	if (unlikely(is_sbi_flag_set(sbi, SBI_POR_DOING)))
		goto redirty_out; // Redirty the page and try again later.

	// Check if the page is fully within the file size, or if fs-verity is active (needs full page),
	// or if it's compressed data (handled in full blocks).
	if (folio->index < end_index ||
			f2fs_verity_in_progress(inode) ||
			compr_blocks)
		goto write; // If yes, proceed to write the page.

	/*
	 * Handle partial pages at the end of the file (EOF).
	 * Calculate the offset of the last valid byte within the page.
	 */
	offset = i_size & (PAGE_SIZE - 1);
	// If the page index is completely beyond the file size (folio->index >= end_index + 1),
	// OR if the file size is exactly page-aligned (offset == 0) and this is the page *at* the end_index,
	// then this page contains no file data (it's a hole or past EOF).
	if ((folio->index >= end_index + 1) || !offset)
		goto out; // Nothing to write, just unlock and return.

	// If we are here, it's the last partial page (folio->index == end_index and offset > 0).
	// Zero out the portion of the folio *after* the end of the file to avoid writing stale data.
	folio_zero_segment(folio, offset, folio_size(folio));

write:
	// Directories and quota files have special handling as their blocks are often managed by checkpoint.
	if (S_ISDIR(inode->i_mode) || quota_inode) {
		/*
		 * For quota files, block allocation might occur. We need to take the
		 * node_write read lock to prevent potential races with discard operations
		 * that could happen during checkpoint.
		 */
		if (quota_inode)
			f2fs_down_read(&sbi->node_write); // Acquire read lock

		// For dir/quota, assume necessary locks (like page lock) are already held or not needed here.
		fio.need_lock = LOCK_DONE;
		// Call the core function to prepare the write operation (find/allocate block).
		err = f2fs_do_write_data_page(&fio);

		// Release the lock if it was taken for a quota inode.
		if (quota_inode)
			f2fs_up_read(&sbi->node_write); // Release read lock

		// Go to common post-write processing.
		goto done;
	}

	// --- Regular file data page handling ---

	// If writeback is not for memory reclaim (e.g., fsync, background flush),
	// set flag to potentially trigger f2fs_balance_fs later.
	if (!wbc->for_reclaim)
		need_balance_fs = true;
	// If writeback *is* for memory reclaim:
	else if (has_not_enough_free_secs(sbi, 0, 0))
		// Check if there's enough free space. If not, don't block reclaim; redirty and try later.
		goto redirty_out;
	else
		// If writing for reclaim and space *is* available, mark inode as having "hot" data.
		set_inode_flag(inode, FI_HOT_DATA);

	// Initialize error to -EAGAIN for the inline data / retry logic below.
	err = -EAGAIN;
	// Check if the inode currently uses inline data (data stored in inode block).
	if (f2fs_has_inline_data(inode)) {
		// Attempt to write the data directly into the inline area.
		err = f2fs_write_inline_data(inode, folio);
		// If inline write succeeded (err == 0), the page is handled.
		if (!err)
			goto out;
		// If inline write failed with -ENOSPC or other error, err holds the code.
		// If it failed with -EAGAIN (e.g., data grew too large), we fall through.
	}

	// If inline write wasn't attempted or failed with -EAGAIN:
	if (err == -EAGAIN) {
		// Attempt the write using normal block allocation. fio.need_lock is LOCK_RETRY initially.
		err = f2fs_do_write_data_page(&fio);
		// If f2fs_do_write_data_page returns -EAGAIN, it means it needed a lock it didn't take.
		if (err == -EAGAIN) {
			// This shouldn't happen for compressed blocks as they should handle locking differently.
			f2fs_bug_on(sbi, compr_blocks);
			// Set the lock requirement to explicitly request locking.
			fio.need_lock = LOCK_REQ;
			// Retry the write operation, this time requesting the necessary lock.
			err = f2fs_do_write_data_page(&fio);
		}
	}

	// Post-write processing for regular files:
	if (err) {
		// If an error occurred during write attempt, tell VFS not to update i_size based on this page.
		file_set_keep_isize(inode);
	} else {
		// If write succeeded, update the last known size written to disk for this inode.
		spin_lock(&F2FS_I(inode)->i_size_lock); // Lock protecting size info.
		// psize is the offset of the end of the current page.
		if (F2FS_I(inode)->last_disk_size < psize)
			F2FS_I(inode)->last_disk_size = psize;
		spin_unlock(&F2FS_I(inode)->i_size_lock); // Release lock.
	}

done:
	// Check for errors after attempting the write (inline or block-based).
	// -ENOENT can happen if blocks were concurrently punched/trimmed, which isn't a write error per se.
	if (err && err != -ENOENT)
		goto redirty_out; // Redirty the page on most errors.

out:
	// Decrement the VFS dirty page counter for this inode.
	inode_dec_dirty_pages(inode);
	// If an error occurred that prevented writing:
	if (err) {
		// Mark the folio as not containing up-to-date data relative to the disk.
		folio_clear_uptodate(folio);
		// Clear any internal F2FS GC flags associated with the page.
		clear_page_private_gcing(page);
	}

	// If this writeback was triggered by memory reclaim:
	if (wbc->for_reclaim) {
		// Conditionally submit any pending merged BIOs for DATA pages.
		f2fs_submit_merged_write_cond(sbi, NULL, page, 0, DATA);
		// Clear the hot data flag now that reclaim write is done (or failed).
		clear_inode_flag(inode, FI_HOT_DATA);
		// If the inode is now clean, remove it from the dirty inode list.
		f2fs_remove_dirty_inode(inode);
		// Don't report submission status upwards during reclaim path.
		submitted = NULL;
	}
	// **Crucial:** Unlock the folio, allowing other operations on it.
	folio_unlock(folio);

	// If it's a regular file subject to quotas, not currently handled by a dedicated
	// writeback task, and balancing is allowed by the caller:
	if (!S_ISDIR(inode->i_mode) && !IS_NOQUOTA(inode) &&
			!F2FS_I(inode)->wb_task && allow_balance)
		// Potentially trigger background cleaning/GC if needed (especially if need_balance_fs was set).
		f2fs_balance_fs(sbi, need_balance_fs);

	// If a checkpoint error occurred *during* this function's execution:
	if (unlikely(f2fs_cp_error(sbi))) {
		// Force submission of any pending merged data writes.
		f2fs_submit_merged_write(sbi, DATA);
		// Force submission of any pending merged IPU (In-Place Update) writes.
		if (bio && *bio)
			f2fs_submit_merged_ipu_write(sbi, bio, NULL);
		// Don't report submission status if errors occurred.
		submitted = NULL;
	}

	// If the caller provided a place to store the submission status:
	if (submitted)
		// Report whether f2fs_do_write_data_page actually submitted a BIO.
		*submitted = fio.submitted;

	// Return 0 indicates the writepage operation completed its logic for this page
	// (even if it resulted in redirtying). The page lock is released.
	return 0;

redirty_out:
	// Mark the folio as dirty again using the VFS helper function.
	// This puts it back on the appropriate list for a later writeback attempt.
	folio_redirty_for_writepage(wbc, folio);
	/*
	 * If there was no specific error (e.g., EAGAIN from locking, low space during reclaim)
	 * OR if we are in the reclaim path, tell the MM layer to simply reactivate
	 * the page (keep it cached). This avoids potentially marking the mapping
	 * with EIO unnecessarily, especially for transient issues like EAGAIN.
	 * AOP_WRITEPAGE_ACTIVATE is the standard return code for this.
	 */
	if (!err || wbc->for_reclaim)
		return AOP_WRITEPAGE_ACTIVATE;

	// **Crucial:** Unlock the folio before returning an error code.
	folio_unlock(folio);
	// Return the specific error code encountered (e.g., -EIO, -ENOSPC) to the caller (MM).
	return err;
}
```

