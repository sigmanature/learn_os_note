```C

// F2FS: Handle truncation within the last (partially kept) compressed cluster.
int f2fs_truncate_partial_cluster(struct inode *inode, u64 from, bool lock)
{
	void *fsdata = NULL; // Buffer for decompressed data or other fs-specific data
	struct page *pagep;  // Pointer to a page (often the compressed data page)
	// Log2 of cluster size in pages
	int log_cluster_size = F2FS_I(inode)->i_log_cluster_size;
	// Page index of the start of the cluster containing 'from'
	pgoff_t start_idx = from >> (PAGE_SHIFT + log_cluster_size) <<
							log_cluster_size;
	int err;

	// Check if the cluster at start_idx is actually compressed.
	// Returns >0 if compressed, 0 if not, <0 on error.
	err = f2fs_is_compressed_cluster(inode, start_idx);
	if (err < 0) // Error checking status
		return err;

	/* truncate normal cluster */
	// If the cluster is not compressed, this function was likely called in error,
	// or it's an edge case. Fall back to standard block truncation for this range.
	if (!err)
		return f2fs_do_truncate_blocks(inode, from, lock);

	/* truncate compressed cluster */
	// Prepare for overwriting the compressed cluster. This typically involves
	// decompressing it into 'fsdata' (which will point to an array of pages).
	// 'pagep' might point to the compressed data page.
	err = f2fs_prepare_compress_overwrite(inode, &pagep,
						start_idx, &fsdata);

	/* should not be a normal cluster */
	// Sanity check: f2fs_prepare_compress_overwrite should not return 0 (normal cluster) here,
	// as f2fs_is_compressed_cluster indicated it was compressed.
	f2fs_bug_on(F2FS_I_SB(inode), err == 0);

	if (err <= 0) // If preparation failed or indicated no action needed
		return err;

	// If err > 0, it means preparation was successful, and fsdata contains decompressed pages.
	if (err > 0) {
		struct page **rpages = fsdata; // fsdata is an array of page pointers (decompressed cluster)
		int cluster_size = F2FS_I(inode)->i_cluster_size; // Number of pages in a cluster
		int i;

		// Iterate backwards through the pages of the decompressed cluster.
		for (i = cluster_size - 1; i >= 0; i--) {
			struct folio *folio = page_folio(rpages[i]); // Get folio for the page
			loff_t start_offset_of_folio = folio->index << PAGE_SHIFT; // Byte offset of this folio

			// If the 'from' offset (where truncation begins) is at or before this folio's start,
			// the entire folio and subsequent parts of it are truncated.
			if (from <= start_offset_of_folio) {
				folio_zero_segment(folio, 0, folio_size(folio)); // Zero the whole folio
			} else {
				// 'from' is within this folio. Zero from ('from' - folio_start_offset) to folio_end.
				folio_zero_segment(folio, from - start_offset_of_folio,
						folio_size(folio));
				break; // All previous folios (i+1 to cluster_size-1) are fully zeroed.
			       // This folio is partially zeroed. Folios before this are untouched.
			}
		}

		// Re-compress the modified (partially zeroed) cluster and write it back.
		// 'true' indicates data was modified.
		f2fs_compress_write_end(inode, fsdata, start_idx, true);
	}
	return 0;
}


```