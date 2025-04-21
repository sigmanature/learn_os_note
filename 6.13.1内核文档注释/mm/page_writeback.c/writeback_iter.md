```C
/**
 * writeback_iter - iterate folio of a mapping for writeback
 * @mapping: address space structure to write
 * @wbc: writeback context
 * @folio: previously iterated folio (%NULL to start)
 * @error: in-out pointer for writeback errors (see below)
 *
 * This function returns the next folio for the writeback operation described by
 * @wbc on @mapping and should be called in a while loop in the ->writepages
 * implementation.
 *
 * To start the writeback operation, %NULL is passed in the @folio argument, and
 * for every subsequent iteration the folio returned previously should be passed
 * back in.
 *
 * If there was an error in the per-folio writeback inside the writeback_iter()
 * loop, @error should be set to the error value before calling writeback_iter()
 * again.
 *
 * Once the writeback described in @wbc has finished, this function will return
 * %NULL and if there was an error in any iteration restore it to @error.
 *
 * Note: callers should not manually break out of the loop using break or goto
 * but must keep calling writeback_iter() until it returns %NULL.
 *
 * Return: the folio to write or %NULL if the loop is done.
 */
struct folio *writeback_iter(struct address_space *mapping,
		struct writeback_control *wbc, struct folio *folio, int *error)
{
	// Check if this is the first iteration (folio is NULL).
	if (!folio) {
		// Initialize the folio batch within the writeback control structure.
		// This is used by writeback_get_folio for efficient page lookup.
		folio_batch_init(&wbc->fbatch);
		// Initialize/reset the saved error status in wbc and the caller's error variable.
		wbc->saved_err = *error = 0;

		/*
		 * Determine the starting index for the writeback scan.
		 */
		// If range_cyclic is set (e.g., background writeback), start from
		// where the last cyclic writeback left off (mapping->writeback_index).
		if (wbc->range_cyclic)
			wbc->index = mapping->writeback_index;
		// Otherwise (e.g., sync, specific range writeback), start from the
		// beginning of the range specified in wbc.
		else
			wbc->index = wbc->range_start >> PAGE_SHIFT;

		/*
		 * For synchronous/integrity writeback (WB_SYNC_ALL) or when explicitly
		 * requested (tagged_writepages), pre-tag all potentially dirty pages
		 * in the range with PAGECACHE_TAG_TOWRITE. This helps prevent
		 * missing pages dirtied during the writeback process itself.
		 */
		if (wbc->sync_mode == WB_SYNC_ALL || wbc->tagged_writepages)
			// tag_pages_for_writeback marks pages in the specified range.
			// wbc_end(wbc) calculates the effective end index based on wbc->range_end.
			tag_pages_for_writeback(mapping, wbc->index,
					wbc_end(wbc));
	} else { // This is a subsequent iteration (folio is not NULL).
		// Decrement the number of pages remaining to be written for non-sync modes.
		wbc->nr_to_write -= folio_nr_pages(folio);

		// Sanity check: the caller should have reset *error if it handled it.
		WARN_ON_ONCE(*error > 0);

		/*
		 * Handle completion conditions based on writeback mode.
		 */
		// For integrity writeback (WB_SYNC_ALL):
		if (wbc->sync_mode == WB_SYNC_ALL) {
			// If an error occurred in the previous iteration (*error is set)
			// and we haven't saved an error yet, save this first error.
			// We continue iterating even after errors to ensure all tagged
			// pages are processed (e.g., to clear state).
			if (*error && !wbc->saved_err)
				wbc->saved_err = *error;
			// Reset *error for the next iteration's input. The final error
			// will be restored from wbc->saved_err at the end.
			*error = 0;
		} else { // For background/non-integrity writeback:
			// If an error occurred OR we have written the requested number
			// of pages (wbc->nr_to_write <= 0), stop the iteration.
			if (*error || wbc->nr_to_write <= 0)
				goto done; // Jump to the cleanup and exit path.
		}
	}

	// Get the next dirty folio to write back using the folio batch and current index.
	// writeback_get_folio finds the next folio marked dirty (or TOWRITE if tagged).
	folio = writeback_get_folio(mapping, wbc);

	// Check if writeback_get_folio returned NULL (no more suitable folios found).
	if (!folio) {
		/*
		 * End of iteration cleanup.
		 */
		// For cyclic writeback, reset the index to 0 if we reached the end
		// of the mapping without finding more pages. This prevents deadlocks
		// related to lock ordering if we immediately looped back to the start.
		// The next cyclic scan will start from 0.
		if (wbc->range_cyclic)
			mapping->writeback_index = 0;

		/*
		 * Restore the first error encountered during integrity sync (if any)
		 * back to the caller's error variable.
		 */
		*error = wbc->saved_err;
	}
	// Return the found folio, or NULL if the iteration is complete.
	return folio;

done: // Exit path for background writeback completion or error.
	// If cyclic writeback, save the index of the *next* folio after the current
	// one, so the next cycle starts there.
	if (wbc->range_cyclic)
		mapping->writeback_index = folio_next_index(folio);
	// Release resources associated with the folio batch.
	folio_batch_release(&wbc->fbatch);
	// Return NULL to signal the end of the iteration to the caller.
	return NULL;
}
```