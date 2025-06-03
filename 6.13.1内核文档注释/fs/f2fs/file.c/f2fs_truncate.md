**相关函数**
[f2fs_truncate_blocks](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/truncate.c/f2fs_truncate_blocks.md)
[f2fs_truncate_partial_cluster](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/truncate.c/f2fs_truncate_partial_cluster.md)
```C
// F2FS specific function to handle block deallocation for truncation.
// Called by f2fs_setattr when file size is reduced.
int f2fs_truncate(struct inode *inode)
{
	int err;

	// Check for filesystem-wide critical errors
	if (unlikely(f2fs_cp_error(F2FS_I_SB(inode))))
		return -EIO;

	// Only proceed for regular files, directories, or symlinks
	if (!(S_ISREG(inode->i_mode) || S_ISDIR(inode->i_mode) ||
				S_ISLNK(inode->i_mode)))
		return 0; // No truncation needed for other types

	trace_f2fs_truncate(inode); // Tracepoint for debugging

	// Fault injection point for testing error paths
	if (time_to_inject(F2FS_I_SB(inode), FAULT_TRUNCATE))
		return -EIO;

	// Ensure quota information is initialized for this inode
	err = f2fs_dquot_initialize(inode);
	if (err)
		return err;

	/* we should check inline_data size */
	// If the inode *cannot* have inline data (e.g., new size is 0, or it's not a candidate)
	// and it currently *is* inline, convert it out of inline mode.
	// Note: i_size is already updated to the new, smaller size.
	if (!f2fs_may_inline_data(inode)) {
		err = f2fs_convert_inline_inode(inode);
		if (err)
			return err;
	}

	// Perform the actual block deallocation starting from the new inode size.
	// The 'true' means acquire necessary locks.
	err = f2fs_truncate_blocks(inode, i_size_read(inode), true);
	if (err)
		return err;

	// Update modification and change times
	inode_set_mtime_to_ts(inode, inode_set_ctime_current(inode));
	// Mark inode as dirty, but 'false' means it doesn't need immediate checkpoint sync
	// because block deallocations are already persistent via their own metadata updates.
	f2fs_mark_inode_dirty_sync(inode, false);
	return 0;
}
```