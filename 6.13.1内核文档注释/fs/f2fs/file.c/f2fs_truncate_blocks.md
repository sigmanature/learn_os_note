```C
// F2FS function to truncate blocks from a given offset.
int f2fs_truncate_blocks(struct inode *inode, u64 from, bool lock)
{
	u64 free_from = from; // The actual offset from which blocks will be freed.
	int err;

#ifdef CONFIG_F2FS_FS_COMPRESSION
	/*
	 * for compressed file, only support cluster size
	 * aligned truncation.
	 */
	// If the file is compressed, round 'from' up to the next cluster boundary.
	// F2FS can only free entire compressed clusters.
	if (f2fs_compressed_file(inode))
		free_from = round_up(from,
				F2FS_I(inode)->i_cluster_size << PAGE_SHIFT);
#endif

	// Call the core block truncation function with the (potentially adjusted) 'free_from' offset.
	err = f2fs_do_truncate_blocks(inode, free_from, lock);
	if (err)
		return err;

#ifdef CONFIG_F2FS_FS_COMPRESSION
	/*
	 * For compressed file, after release compress blocks, don't allow write
	 * direct, but we should allow write direct after truncate to zero.
	 */
	// If compressed file and truncated to zero size, clear the FI_COMPRESS_RELEASED flag.
	// This flag might prevent direct writes if blocks were previously released.
	if (f2fs_compressed_file(inode) && !free_from // free_from is 0 if 'from' was 0
			&& is_inode_flag_set(inode, FI_COMPRESS_RELEASED))
		clear_inode_flag(inode, FI_COMPRESS_RELEASED);

	// If 'from' was not cluster-aligned for a compressed file, 'free_from' is larger.
	// This means the last cluster containing 'from' was not fully freed by f2fs_do_truncate_blocks.
	// We need to handle this partial cluster truncation.
	if (from != free_from) {
		err = f2fs_truncate_partial_cluster(inode, from, lock);
		if (err)
			return err;
	}
#endif

	return 0;
}
```