**函数调用流程图**
```mermaid
graph TD
subgraph a[**f2fs_cluster_blocks_are_contiguous**]	
end
subgraph b[**f2fs_update_read_extent_tree_range_compressed**]
A[__set_extent_info]
end
```
```C
#ifdef CONFIG_F2FS_FS_COMPRESSION // Compile this section only if compression is enabled

/**
 * f2fs_update_read_extent_tree_range_compressed - Update extent cache for compressed files.
 * @inode: Inode of the compressed file.
 * @fofs: Starting file offset (cluster-aligned).
 * @blkaddr: Starting block address of the compressed data.
 * @llen: Length of the extent unit (should be cluster size).
 * @c_len: Number of contiguous clusters starting at fofs.
 *
 * This function adds or merges extent information for a range of compressed
 * clusters into the inode's read extent tree. It's typically called when
 * accessing compressed data on a read-only filesystem.
 */
void f2fs_update_read_extent_tree_range_compressed(struct inode *inode,
				pgoff_t fofs, block_t blkaddr, unsigned int llen,
				unsigned int c_len)
{
	struct f2fs_sb_info *sbi = F2FS_I_SB(inode); // Get SB info
	// Get the read extent tree associated with this inode
	struct extent_tree *et = F2FS_I(inode)->extent_tree[EX_READ];
	struct extent_node *en = NULL; // Found/merged extent node
	struct extent_node *prev_en = NULL, *next_en = NULL; // Adjacent nodes
	struct extent_info ei; // Structure to hold new extent info
	struct rb_node **insert_p = NULL, *insert_parent = NULL; // For RB tree insertion
	bool leftmost = false; // Flag for RB tree insertion hint

	// Tracepoint for debugging/monitoring
	trace_f2fs_update_read_extent_tree_range(inode, fofs, llen,
						blkaddr, c_len);

	/* it is safe here to check FI_NO_EXTENT w/o et->lock in ro image */
	// Optimization: If inode is marked as having no extents, skip update.
	// Checking without lock is safe on read-only mounts.
	if (is_inode_flag_set(inode, FI_NO_EXTENT))
		return;

	// Acquire write lock to modify the extent tree (RB-tree and list)
	write_lock(&et->lock);

	// Look for an existing extent node covering the starting offset 'fofs'.
	// Also finds potential neighbors (prev_en, next_en) for merging and
	// determines the insertion point (insert_p, insert_parent) if needed.
	en = __lookup_extent_node_ret(&et->root,
					et->cached_en, fofs,
					&prev_en, &next_en,
					&insert_p, &insert_parent,
					&leftmost);
	// If an extent already exists exactly at 'fofs', assume it's up-to-date and do nothing.
	// Merging handles cases where adjacent extents need combining.
	if (en)
		goto unlock_out;

	// Prepare the extent information structure for the new extent.
	// 'true' for keep_clen ensures ei.c_len is preserved.
	__set_extent_info(&ei, fofs, llen, blkaddr, true, 0, 0, EX_READ);
	//ei的c_len是压缩簇的extent_info的核心字段。它代表的是压缩簇中块的个数。
	ei.c_len = c_len;

	// Attempt to merge the new extent info 'ei' with its neighbors (prev_en, next_en).
	// If merging succeeds, __try_merge_extent_node returns the merged node.
	if (!__try_merge_extent_node(sbi, et, &ei, prev_en, next_en))
		// If merging is not possible or fails, insert 'ei' as a new node in the extent tree.
		__insert_extent_tree(sbi, et, &ei,
				insert_p, insert_parent, leftmost);
unlock_out:
	// Release the write lock
	write_unlock(&et->lock);
}
#endif // CONFIG_F2FS_FS_COMPRESSION
```