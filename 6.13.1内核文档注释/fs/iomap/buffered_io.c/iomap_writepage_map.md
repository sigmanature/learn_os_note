**相关函数**
* [iomap_writepage_map_blocks](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/iomap/buffered_io.c/iomap_writepage_map_blocks.md)
```C
/**
 * iomap_writepage_map - Map and prepare a single folio for iomap writeback.
 * @wpc: Context structure for iomap writepage operations.
 * @wbc: Writeback control structure.
 * @folio: The folio to process.
 *
 * This function takes a folio identified by writeback_iter, handles EOF conditions,
 * finds dirty ranges within the folio, maps them to disk blocks using the
 * filesystem's map_blocks callback, and adds the mapped extents to the pending
 * I/O context (wpc) via iomap_add_to_ioend. It also manages the folio's
 * writeback state.
 *
 * Return: 0 on success, or a negative error code on failure.
 */
static int iomap_writepage_map(struct iomap_writepage_ctx *wpc,
		struct writeback_control *wbc, struct folio *folio)
{
	// Get the per-folio state if it exists (used for multi-block folios).
	struct iomap_folio_state *ifs = folio->private;
	// Get the inode associated with the folio's mapping.
	struct inode *inode = folio->mapping->host;
	// Calculate the starting byte position of the folio within the file.
	u64 pos = folio_pos(folio);
	// Calculate the ending byte position of the folio within the file.
	u64 end_pos = pos + folio_size(folio);
	// Variable to store the block-aligned end position.
	u64 end_aligned = 0;
	// Counter for the number of blocks successfully mapped and added for I/O.
	unsigned count = 0;
	// Variable to store errors encountered during mapping.
	int error = 0;
	// Variable to store the length of the current dirty range found.
	u32 rlen;

	// Sanity checks: Folio should be locked, not dirty (cleared by caller),
	// and not already under writeback when entering this function.
	WARN_ON_ONCE(!folio_test_locked(folio));
	WARN_ON_ONCE(folio_test_dirty(folio)); // writeback_iter should have cleared dirty
	WARN_ON_ONCE(folio_test_writeback(folio)); // Should not be under writeback yet

	// Tracepoint for writepage operation.
	trace_iomap_writepage(inode, pos, folio_size(folio));

	// Handle interaction with the end-of-file (EOF).
	// This checks if the folio is beyond EOF or straddles it.
	// If it straddles, it zeros the part beyond EOF and adjusts end_pos.
	// If it's entirely beyond EOF, it returns false.
	if (!iomap_writepage_handle_eof(folio, inode, &end_pos)) {
		folio_unlock(folio); // Unlock the folio as we're skipping it.
		return 0; // Return success (folio handled by skipping).
	}
	// Sanity check: end_pos should be greater than pos after EOF handling.
	WARN_ON_ONCE(end_pos <= pos);

	// Check if the folio potentially spans multiple filesystem blocks.
	if (i_blocks_per_folio(inode, folio) > 1) {
		// If it does and no per-folio state exists yet:
		if (!ifs) {
			// Allocate the iomap_folio_state structure.
			ifs = ifs_alloc(inode, folio, 0);
			// Mark the entire relevant range within the folio as dirty
			// in the internal tracking (likely a bitmask in ifs).
			iomap_set_range_dirty(folio, 0, end_pos - pos);
		}

		/*
		 * For multi-block folios, the writeback bit needs to stay set
		 * until *all* blocks have completed I/O. We achieve this by
		 * adding a bias (incrementing) to the pending write counter.
		 * Each block completion decrements it, and the final decrement
		 * (removing the bias after submission loop) triggers clearing
		 * the writeback bit if all blocks are done.
		 */
		// Sanity check: pending count should be 0 before adding bias.
		WARN_ON_ONCE(atomic_read(&ifs->write_bytes_pending) != 0);
		// Add the bias.
		atomic_inc(&ifs->write_bytes_pending);
	}

	/*
	 * Set the folio's writeback flag. This must be done early, especially
	 * for single-block folios, as I/O completion could happen very quickly
	 * after submission, before we finish processing the folio here.
	 */
	folio_start_writeback(folio);

	/*
	 * Iterate through the folio, finding contiguous ranges of dirty bytes
	 * that need to be written.
	 */
	// Calculate the filesystem block-aligned end position for searching dirty ranges.
	end_aligned = round_up(end_pos, i_blocksize(inode));
	// Loop while iomap_find_dirty_range finds a dirty range (rlen > 0).
	// 'pos' is updated by the loop to the start of the next search area.
	while ((rlen = iomap_find_dirty_range(folio, &pos, end_aligned))) {
		// For each dirty range found, map it to physical blocks.
		error = iomap_writepage_map_blocks(wpc, wbc, folio, inode,
				pos, end_pos, rlen, &count);
		// If mapping failed for any part of the folio, stop processing it.
		if (error)
			break;
		// Advance position past the just-processed range.
		pos += rlen;
	}

	// If any blocks were successfully mapped and added for I/O, increment folio count.
	if (count)
		wpc->nr_folios++;

	/*
	 * Clear the internal dirty tracking bits for the entire folio now that
	 * we have processed all relevant ranges. This handles cases like partial
	 * folios at EOF where dirty bits might exist beyond the mapped area.
	 */
	iomap_clear_range_dirty(folio, 0, folio_size(folio));

	/*
	 * Unlock the folio before potentially clearing the writeback bit.
	 */
	folio_unlock(folio);

	/*
	 * Handle clearing the writeback bit. This is usually done by the I/O
	 * completion handler (bio_endio). However, if no blocks were actually
	 * submitted ('count' is 0), or if it was a multi-block folio and all
	 * its block I/Os completed *before* we got here, we need to clear it manually.
	 */
	if (ifs) { // Multi-block folio case
		// Decrement the pending count (removing the bias). If this makes the
		// count zero, it means all blocks for this folio have completed I/O.
		if (atomic_dec_and_test(&ifs->write_bytes_pending))
			// All blocks done, clear the writeback flag.
			folio_end_writeback(folio);
	} else { // Single-block folio case (or no ifs allocated)
		// If no blocks were submitted for I/O ('count' is 0), clear writeback.
		// If blocks *were* submitted, the I/O completion handler is responsible.
		if (!count)
			folio_end_writeback(folio);
	}

	// If an error occurred during mapping, record it in the address space.
	// Note: This sets AS_EIO, which might be too coarse for some errors.
	if (error)
	    mapping_set_error(inode->i_mapping, error);
	// Return the error status from mapping this folio.
	return error;
}
```
