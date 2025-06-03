```C
void truncate_inode_pages(struct address_space *mapping, loff_t lstart)
{
	// Call the ranged version, truncating from lstart to the end of the file.
	truncate_inode_pages_range(mapping, lstart, (loff_t)-1);
}
EXPORT_SYMBOL(truncate_inode_pages);

/**
 * truncate_inode_pages_range - truncate range of pages specified by start & end byte offsets
 * @mapping: mapping to truncate
 * @lstart: offset from which to truncate
 * @lend: offset to which to truncate (inclusive)
 *
 * Truncate the page cache, removing the pages that are between
 * specified offsets (and zeroing out partial pages
 * if lstart or lend + 1 is not page aligned).
 *
 * Truncate takes two passes - the first pass is nonblocking.  It will not
 * block on page locks and it will not block on writeback.  The second pass
 * will wait.  This is to prevent as much IO as possible in the affected region.
 * The first pass will remove most pages, so the search cost of the second pass
 * is low.
 *
 * We pass down the cache-hot hint to the page freeing code.  Even if the
 * mapping is large, it is probably the case that the final pages are the most
 * recently touched, and freeing happens in ascending file offset order.
 *
 * Note that since ->invalidate_folio() accepts range to invalidate
 * truncate_inode_pages_range is able to handle cases where lend + 1 is not
 * page aligned properly.
 */
void truncate_inode_pages_range(struct address_space *mapping,
				loff_t lstart, loff_t lend)
{
	pgoff_t		start;		/* inclusive page index for full truncation */
	pgoff_t		end;		/* exclusive page index for full truncation */
	struct folio_batch fbatch;  // Batch of folios for efficient processing
	pgoff_t		indices[PAGEVEC_SIZE]; // Buffer for page indices
	pgoff_t		index;      // Current page index being processed
	int		i;
	struct folio	*folio;
	bool		same_folio; // True if lstart and lend are in the same folio

	if (mapping_empty(mapping)) // If no pages in mapping, nothing to do
		return;

	/*
	 * 'start' and 'end' always covers the range of pages to be fully
	 * truncated. Partial pages are covered with 'partial_start' at the
	 * start of the range and 'partial_end' at the end of the range.
	 * Note that 'end' is exclusive while 'lend' is inclusive.
	 */
	// Calculate starting page index for full page removal (rounded up).
	start = (lstart + PAGE_SIZE - 1) >> PAGE_SHIFT;
	if (lend == -1) // -1 means truncate to end of file
		end = -1; // Represents the highest possible page index
	else
		// Calculate ending page index for full page removal (page after lend, exclusive).
		end = (lend + 1) >> PAGE_SHIFT;

	folio_batch_init(&fbatch); // Initialize folio batch
	index = start;
	// First pass (non-blocking): Find and lock pages, remove exceptional entries.
	while (index < end && find_lock_entries(mapping, &index, end - 1,
			&fbatch, indices)) {
		// Handle exceptional entries (e.g. DAX, shadow entries) in the batch
		truncate_folio_batch_exceptionals(mapping, &fbatch, indices);
		// For regular folios, prepare them for cleanup (e.g., wait for I/O if needed, but this pass is non-blocking)
		for (i = 0; i < folio_batch_count(&fbatch); i++)
			truncate_cleanup_folio(fbatch.folios[i]); // Marks clean, waits for writeback (light attempt)
		// Remove the batch of folios from page cache.
		delete_from_page_cache_batch(mapping, &fbatch);
		// Unlock all folios in the batch.
		for (i = 0; i < folio_batch_count(&fbatch); i++)
			folio_unlock(fbatch.folios[i]);
		folio_batch_release(&fbatch); // Release the batch
		cond_resched(); // Yield if needed
	}

	// Handle partial truncation for the folio containing lstart.
	same_folio = (lstart >> PAGE_SHIFT) == (lend >> PAGE_SHIFT);
	// Get and lock the folio containing lstart. FGP_LOCK attempts to lock.
	folio = __filemap_get_folio(mapping, lstart >> PAGE_SHIFT, FGP_LOCK, 0);
	if (!IS_ERR(folio)) { // If folio was found and locked
		// Check if lend is within this same folio after potential splits.
		same_folio = lend < folio_pos(folio) + folio_size(folio);
		// Perform partial truncation on this folio.
		// Returns false if splitting failed and folio shouldn't be entirely discarded.
		if (!truncate_inode_partial_folio(folio, lstart, lend)) {
			// If partial truncation indicates the folio remains (e.g. split failed),
			// adjust 'start' index for full truncation to skip this folio.
			start = folio_next_index(folio);
			if (same_folio)
				// If lend was also in this folio, adjust 'end' as well.
				end = folio->index;
		}
		folio_unlock(folio);
		folio_put(folio);
		folio = NULL;
	}

	// Handle partial truncation for the folio containing lend, if different from lstart's folio.
	if (!same_folio) {
		folio = __filemap_get_folio(mapping, lend >> PAGE_SHIFT,
						FGP_LOCK, 0);
		if (!IS_ERR(folio)) {
			if (!truncate_inode_partial_folio(folio, lstart, lend))
				// If partial truncation indicates folio remains, adjust 'end'.
				end = folio->index;
			folio_unlock(folio);
			folio_put(folio);
		}
	}

	// Second pass (blocking): Find remaining pages, wait for writeback, and remove them.
	index = start;
	while (index < end) {
		cond_resched();
		// Find and get references to pages in the range.
		if (!find_get_entries(mapping, &index, end - 1, &fbatch,
				indices)) {
			// If no pages found from 'start' onwards, we're done.
			if (index == start)
				break;
			// Otherwise, some pages might have been added; restart scan from 'start'.
			index = start;
			continue;
		}

		for (i = 0; i < folio_batch_count(&fbatch); i++) {
			struct folio *folio = fbatch.folios[i];

			if (xa_is_value(folio)) // Skip shadow/exceptional entries handled elsewhere
				continue;

			folio_lock(folio); // Lock the folio
			VM_BUG_ON_FOLIO(!folio_contains(folio, indices[i]), folio); // Sanity check
			folio_wait_writeback(folio); // Wait for any ongoing writeback to complete
			truncate_inode_folio(mapping, folio); // Perform truncation actions on the folio (e.g. remove from LRU)
			folio_unlock(folio);
		}
		// Handle any exceptional entries that might have appeared.
		truncate_folio_batch_exceptionals(mapping, &fbatch, indices);
		folio_batch_release(&fbatch); // Release the batch (puts folios)
	}
}
EXPORT_SYMBOL(truncate_inode_pages_range);

/*
 * Handle partial folios.  The folio may be entirely within the
 * range if a split has raced with us.  If not, we zero the part of the
 * folio that's within the [start, end] range, and then split the folio if
 * it's large.  split_page_range() will discard pages which now lie beyond
 * i_size, and we rely on the caller to discard pages which lie within a
 * newly created hole.
 *
 * Returns false if splitting failed so the caller can avoid
 * discarding the entire folio which is stubbornly unsplit.
 */
bool truncate_inode_partial_folio(struct folio *folio, loff_t start, loff_t end)
{
	loff_t pos = folio_pos(folio); // Starting byte offset of the folio
	unsigned int offset, length;   // Offset within folio and length of truncation

	// Calculate offset within the folio where truncation begins.
	if (pos < start)
		offset = start - pos;
	else
		offset = 0;

	// Calculate length of the segment to be truncated within this folio.
	length = folio_size(folio);
	if (pos + length <= (u64)end) // If folio is entirely before or at 'end'
		length = length - offset; // Truncate from 'offset' to folio end
	else
		// Folio straddles 'end', truncate up to 'end'.
		length = end + 1 - pos - offset;

	folio_wait_writeback(folio); // Wait for any I/O on this folio to complete.
	if (length == folio_size(folio)) { // If the entire folio is being truncated
		truncate_inode_folio(folio->mapping, folio); // Full truncation actions
		return true; // Successfully handled
	}

	/*
	 * We may be zeroing pages we're about to discard, but it avoids
	 * doing a complex calculation here, and then doing the zeroing
	 * anyway if the page split fails.
	 */
	// If the mapping is accessible (not a DAX conflict or similar)
	if (!mapping_inaccessible(folio->mapping))
		// Zero the part of the folio that is being truncated.
		folio_zero_range(folio, offset, length);

	// If the folio needs explicit invalidation (e.g., for DAX or other special mappings)
	if (folio_needs_release(folio))
		folio_invalidate(folio, offset, length); // Invalidate the specified range

	if (!folio_test_large(folio)) // If not a large folio, no split needed.
		return true;
	if (split_folio(folio) == 0) // Attempt to split the large folio. Returns 0 on success.
		return true;
	// Split failed.
	if (folio_test_dirty(folio)) // If it's dirty and split failed, we can't easily discard.
		return false; // Indicate failure to split a dirty folio.
	// If not dirty (or became clean), or split succeeded, proceed to truncate.
	truncate_inode_folio(folio->mapping, folio); // Truncate the (now smaller or original) folio.
	return true;
}
```
