 ```c
/**
 * f2fs_do_sync_file - Flush all dirty data and metadata pages of a file
 * @file:	File structure representing the open file.
 * @start:	Start offset for the range to sync (inclusive).
 * @end:	End offset for the range to sync (inclusive).
 * @datasync:	Non-zero if this is for fdatasync() (data and essential metadata).
 *              Zero if this is for fsync() (data and all metadata).
 * @atomic:	Indicates if atomic write semantics should be enforced for node pages.
 *
 * This function is the core implementation for fsync() and fdatasync() operations
 * in F2FS. It ensures that modified data and/or metadata for the given file
 * within the specified range are written to the underlying storage device.
 *
 * Returns:
 * 0 on success, or a negative error code on failure.
 */
static int f2fs_do_sync_file(struct file *file, loff_t start, loff_t end,
			     int datasync, bool atomic)
{
	// Get the inode associated with the file. The address space mapping links the file to its inode.
	struct inode *inode = file->f_mapping->host;
	// Get the F2FS superblock information from the inode. Contains filesystem-wide data and options.
	struct f2fs_sb_info *sbi = F2FS_I_SB(inode);
	// Get the inode number.
	nid_t ino = inode->i_ino;
	// Initialize return value to 0 (success).
	int ret = 0;
	// Variable to store the reason for triggering a checkpoint, if needed. Initialized to 0 (no reason yet).
	enum cp_reason_type cp_reason = 0;
	// Initialize writeback control structure for managing the writeback process.
	struct writeback_control wbc = {
		.sync_mode = WB_SYNC_ALL,	// Sync mode: wait for completion.
		.nr_to_write = LONG_MAX,	// Number of pages to write: effectively infinite (write all).
		.for_reclaim = 0,		// Not for memory reclaim.
	};
	// Sequence ID used for tracking batches of node page writeback operations.
	unsigned int seq_id = 0;

	// If the filesystem is mounted read-only, there's nothing to sync. Return success immediately.
	if (unlikely(f2fs_readonly(inode->i_sb)))
		return 0;

	// Trace function entry for debugging and performance analysis.
	trace_f2fs_sync_file_enter(inode);

	// Directories typically only have metadata changes (inode, dentry pages).
	// Data writeback logic is skipped, go directly to metadata handling.
	if (S_ISDIR(inode->i_mode))
		goto go_write;

	// Optimization: If this is fdatasync() OR if the number of dirty pages is very small,
	// try to perform In-Place Updates (IPU) for data blocks. This can be more efficient
	// than allocating new blocks for small overwrites.
	// Set the FI_NEED_IPU flag to indicate this preference during writeback.
	if (datasync || get_dirty_pages(inode) <= SM_I(sbi)->min_fsync_blocks)
		set_inode_flag(inode, FI_NEED_IPU);
	
	// Write back all dirty data pages within the specified range [start, end] and wait for completion.
	// This function handles the actual data writeback.
	ret = file_write_and_wait_range(file, start, end);
	
	// Clear the FI_NEED_IPU flag now that data writeback is done.
	clear_inode_flag(inode, FI_NEED_IPU);

	// Check if file_write_and_wait_range failed or if checkpoints are globally disabled.
	if (ret || is_sbi_flag_set(sbi, SBI_CP_DISABLED)) {
		// Trace function exit with the current status.
		trace_f2fs_sync_file_exit(inode, cp_reason, datasync, ret);
		// Return the error code or 0 if checkpoints are disabled (as we can't guarantee durability).
		return ret;
	}

	// Check if the inode's metadata needs to be written.
	// f2fs_skip_inode_update returns true for datasync unless the inode itself is dirty.
	// For fsync(), it always returns false, forcing an inode write.
	if (!f2fs_skip_inode_update(inode, datasync)) {
		// Write the inode block to disk. NULL indicates default writeback control.
		f2fs_write_inode(inode, NULL);
		// Proceed to handle node page syncing and potential checkpoint.
		goto go_write;
	}

	/*
	 * Optimization: If there's no record of recent append writes (FI_APPEND_WRITE flag)
	 * AND no persistent record of written data for recovery (f2fs_exist_written_data),
	 * we might be able to skip further expensive operations like checkpoints or flushes.
	 */
	if (!is_inode_flag_set(inode, FI_APPEND_WRITE) &&
	    !f2fs_exist_written_data(sbi, ino, APPEND_INO)) {
		// However, even without append writes, the inode page might have been dirtied
		// just before fsync was called (e.g., timestamp update). Check if it needs writing.
		if (need_inode_page_update(sbi, ino))
			goto go_write; // If inode needs update, proceed to write metadata.

		// Check for update writes (overwrites) using the FI_UPDATE_WRITE flag or persistent recovery info.
		if (is_inode_flag_set(inode, FI_UPDATE_WRITE) ||
		    f2fs_exist_written_data(sbi, ino, UPDATE_INO))
			// If there were update writes, we need to ensure metadata/nodes are synced and potentially flush.
			goto flush_out; 
		// If no append writes, no update writes, and inode page is clean, nothing more to do.
		goto out;
	} else {
		/*
		 * Handle Over-Provisioned Update (OPU) case with strict fsync mode.
		 * In strict mode (FSYNC_MODE_STRICT), especially on devices without reliable write barriers,
		 * node pages (metadata) could be written before the corresponding data pages, leading to
		 * corruption after a sudden power-off (SPO).
		 * To prevent this, force atomic write semantics for node pages if not already requested.
		 * Atomic writes use mechanisms (like node chains) to ensure ordering or atomicity.
		 */
		if (F2FS_OPTION(sbi).fsync_mode == FSYNC_MODE_STRICT && !atomic)
			atomic = true;
	}

go_write:
	/*
	 * Both fdatasync() and fsync() need to be recoverable after power loss.
	 * Check if a full filesystem checkpoint is necessary for this inode.
	 * This usually happens if critical metadata structures were modified.
	 * Acquire read lock on inode's semaphore to safely check checkpoint conditions.
	 */
	f2fs_down_read(&F2FS_I(inode)->i_sem);
	cp_reason = need_do_checkpoint(inode);
	f2fs_up_read(&F2FS_I(inode)->i_sem);

	// If a checkpoint is needed (cp_reason is non-zero):
	if (cp_reason) {
		// Trigger a full filesystem synchronization (checkpoint). This flushes all dirty
		// metadata (including nodes, SIT, NAT) across the entire filesystem.
		// The '1' argument indicates it should wait for completion.
		ret = f2fs_sync_fs(inode->i_sb, 1);

		/*
		 * After a successful checkpoint, the filesystem state is consistent.
		 * The pino (parent inode number) optimization might need adjustment.
		 * Try to fix the pino cache for this inode if necessary.
		 */
		try_to_fix_pino(inode);
		// Since the checkpoint flushed everything related to this inode's recovery,
		// clear the temporary flags indicating recent writes.
		clear_inode_flag(inode, FI_APPEND_WRITE);
		clear_inode_flag(inode, FI_UPDATE_WRITE);
		// Checkpoint handled everything, so we can exit.
		goto out;
	}

sync_nodes:
	// If no checkpoint was needed, we still need to sync the dirty node pages associated with this inode.
	// Increment the counter for active node sync requests.
	atomic_inc(&sbi->wb_sync_req[NODE]);
	// Call the function to write back dirty node pages for this specific inode.
	// Pass the writeback control, atomic flag, and a pointer to store the sequence ID.
	ret = f2fs_fsync_node_pages(sbi, inode, &wbc, atomic, &seq_id);
	// Decrement the counter for active node sync requests.
	atomic_dec(&sbi->wb_sync_req[NODE]);
	// If syncing node pages failed, exit.
	if (ret)
		goto out;

	// Check if the filesystem encountered a checkpoint error. This prevents an infinite loop
	// if f2fs_need_inode_block_update keeps returning true due to CP errors.
	if (unlikely(f2fs_cp_error(sbi))) {
		ret = -EIO; // Report I/O error.
		goto out;
	}

	// Writing node pages might have dirtied the inode block itself (e.g., updating block addresses).
	// Check if the inode block needs to be written again.
	if (f2fs_need_inode_block_update(sbi, ino)) {
		// Mark the inode as dirty and needing synchronous write.
		f2fs_mark_inode_dirty_sync(inode, true);
		// Write the inode block again.
		f2fs_write_inode(inode, NULL);
		// Since the inode write might dirty more node pages (e.g., if it caused block allocation),
		// loop back to sync node pages again to ensure everything is flushed.
		goto sync_nodes;
	}

	/*
	 * If atomic writes were used:
	 * The ordering is guaranteed by the node chain structure used in atomic writes.
	 * If any node write fails or is reordered, the chain will appear broken during recovery,
	 * preventing partial recovery of the fsync operation. We don't need to explicitly wait
	 * for I/O completion here because the structure itself ensures consistency.
	 *
	 * If atomic writes were NOT used:
	 * We need to explicitly wait for the node page writeback operations (identified by seq_id)
	 * to complete before proceeding. This ensures node metadata hits the disk.
	 */
	if (!atomic) {
		ret = f2fs_wait_on_node_pages_writeback(sbi, seq_id);
		// If waiting failed, exit.
		if (ret)
			goto out;
	}

	// Node pages related to append writes are now safely on disk (or guaranteed by atomic write structure).
	// Remove the inode number from the list tracking inodes with recent append writes needing recovery.
	f2fs_remove_ino_entry(sbi, ino, APPEND_INO);
	// Clear the temporary flag.
	clear_inode_flag(inode, FI_APPEND_WRITE);

flush_out:
	// If not using atomic writes AND the fsync mode requires barriers (i.e., not FSYNC_MODE_NOBARRIER),
	// issue a flush command to the underlying block device.
	// This ensures that all data and metadata written so far are actually persisted in non-volatile storage,
	// forcing it out of any hardware caches. Atomic writes often imply barrier semantics or ordering guarantees
	// that make an explicit flush less critical here.
	if (!atomic && F2FS_OPTION(sbi).fsync_mode != FSYNC_MODE_NOBARRIER)
		ret = f2fs_issue_flush(sbi, inode->i_ino);

	// If the flush was successful (or not needed/performed):
	if (!ret) {
		// Remove the inode number from lists tracking inodes needing recovery for update writes or flushes.
		f2fs_remove_ino_entry(sbi, ino, UPDATE_INO);
		clear_inode_flag(inode, FI_UPDATE_WRITE);
		f2fs_remove_ino_entry(sbi, ino, FLUSH_INO);
	}
	// Update filesystem timestamps (e.g., last sync time). REQ_TIME indicates a user-requested time update.
	f2fs_update_time(sbi, REQ_TIME);

out:
	// Trace function exit, recording the outcome.
	trace_f2fs_sync_file_exit(inode, cp_reason, datasync, ret);
	// Return the final status code (0 for success, negative error code for failure).
	return ret;
}
```

