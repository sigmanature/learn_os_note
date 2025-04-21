```C
/**
 * iomap_writepage_map_blocks - Map a contiguous dirty range within a folio.
 * @wpc: Context structure for iomap writepage operations.
 * @wbc: Writeback control structure.
 * @folio: The folio containing the dirty range.
 * @inode: The corresponding inode.
 * @pos: Starting byte position of the dirty range within the file.
 * @end_pos: The effective end position of the folio (adjusted for EOF).
 * @dirty_len: Length of the contiguous dirty range starting at 'pos'.
 * @count: In/Out pointer to a counter for successfully mapped blocks.
 *
 * Takes a specific dirty range within a folio, calls the filesystem's
 * map_blocks operation (potentially multiple times if the mapping is fragmented),
 * and adds the resulting mapped extents (excluding holes) to the pending I/O
 * context using iomap_add_to_ioend.
 *
 * Return: 0 on success, or a negative error code if mapping fails.
 */
static int iomap_writepage_map_blocks(struct iomap_writepage_ctx *wpc,
		struct writeback_control *wbc, struct folio *folio,
		struct inode *inode, u64 pos, u64 end_pos,
		unsigned dirty_len, unsigned *count)
{
	int error = 0; // Initialize error status.

	// Loop as long as there's a remaining portion of the dirty range to map
	// and no error has occurred.
	do {
		// Length of the current mapping provided by map_blocks.
		unsigned map_len;

		// Call the filesystem-specific map_blocks operation.
		// This fills wpc->iomap with details about the block mapping
		// starting at 'pos' for up to 'dirty_len' bytes.
		error = wpc->ops->map_blocks(wpc, inode, pos, dirty_len);
		// If map_blocks failed, break out of the loop.
		if (error)
			break;
		// Tracepoint for the mapping result.
		trace_iomap_writepage_map(inode, pos, dirty_len, &wpc->iomap);

		// Calculate the actual length covered by the current mapping result,
		// capped by the remaining dirty_len.
		map_len = min_t(u64, dirty_len,
			wpc->iomap.offset + wpc->iomap.length - pos);
		// Sanity check: If not using per-folio state (ifs), a single mapping
		// should cover the entire dirty range requested in one go.
		WARN_ON_ONCE(!folio->private && map_len < dirty_len);

		// Process the mapping based on its type.
		switch (wpc->iomap.type) {
		case IOMAP_INLINE:
			// Inline data writeback shouldn't typically happen via writepages.
			WARN_ON_ONCE(1); // Warn if it does.
			error = -EIO; // Treat as an I/O error.
			break;
		case IOMAP_HOLE:
			// This part of the range is unallocated (a hole).
			// No data needs to be written, just skip this extent.
			break;
		default: // Mapped block (IOMAP_MAPPED, IOMAP_UNWRITTEN, etc.)
			// Add this mapped extent to the pending I/O structure (ioend).
			// iomap_add_to_ioend handles BIO creation/merging.
			error = iomap_add_to_ioend(wpc, wbc, folio, inode, pos,
					end_pos, map_len);
			// If adding to ioend succeeded, increment the submitted block count.
			if (!error)
				(*count)++;
			break;
		}
		// Reduce the remaining dirty length.
		dirty_len -= map_len;
		// Advance the position within the file/folio.
		pos += map_len;
	} while (dirty_len && !error); // Continue if more dirty range left and no error.

	/*
	 * If an error occurred during the mapping loop, inform the filesystem
	 * via the discard_folio callback (if provided). This allows the FS
	 * to potentially clean up state related to the failed portion.
	 * 'pos' will indicate the position where the mapping started to fail.
	 */
	if (error && wpc->ops->discard_folio)
		wpc->ops->discard_folio(folio, pos);
	// Return the final error status.
	return error;
}
```