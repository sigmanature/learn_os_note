**相关函数**
*[writeback_iter](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/page_writeback.c/writeback_iter.md)
* [iomap_writepage_map_blocks](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/iomap/buffered_io.c/iomap_writepage_map_blocks.md)
* [iomap_write_pages_handle_eof](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/iomap/buffered_io.c/iomap_write_pages_handle_eof.md)
* [iomap_writepage_map](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/iomap/buffered_io.c/iomap_writepage_map.md)
```C
/**
 * iomap_writepages - Write back dirty pages for a mapping using iomap.
 * @mapping: Address space structure containing the pages to write.
 * @wbc: Writeback control structure defining the scope and mode of writeback.
 * @wpc: Context structure for iomap writepage operations, holding state like
 *       pending BIOs and filesystem callbacks.
 * @ops: Filesystem-specific callbacks for mapping blocks and handling I/O.
 *
 * This is the main entry point for filesystems using the iomap framework to
 * write back dirty pages. It iterates through dirty folios using writeback_iter()
 * and processes each folio using iomap_writepage_map(). Finally, it submits
 * any pending I/O via iomap_submit_ioend().
 *
 * Return: 0 on success, or a negative error code on failure.
 */
int
iomap_writepages(struct address_space *mapping, struct writeback_control *wbc,
		struct iomap_writepage_ctx *wpc,
		const struct iomap_writeback_ops *ops)
{
	struct folio *folio = NULL; // Initialize folio pointer to NULL for the first iteration
	int error; // Variable to store errors encountered during iteration/mapping

	/*
	 * Writeback initiated directly from memory reclaim context (PF_MEMALLOC set,
	 * but not by kswapd) is problematic as it can lead to deadlocks (e.g.,
	 * trying to allocate memory while holding locks needed for allocation).
	 * Warn if this potentially dangerous situation occurs and return an error.
	 */
	if (WARN_ON_ONCE((current->flags & (PF_MEMALLOC | PF_KSWAPD)) ==
			PF_MEMALLOC))
		return -EIO; // Return I/O error to prevent potential deadlock

	// Store the filesystem-specific operations in the writepage context.
	wpc->ops = ops;
	// Loop through dirty folios provided by writeback_iter.
	// writeback_iter handles finding the next folio based on wbc parameters
	// and manages the overall iteration state (range, errors, etc.).
	// The loop continues as long as writeback_iter returns a valid folio.
	// 'error' is passed by reference to writeback_iter to retrieve the final error status.
	while ((folio = writeback_iter(mapping, wbc, folio, &error)))
		// For each folio returned, call iomap_writepage_map to handle
		// mapping its dirty portions to disk blocks and preparing I/O.
		// The error from mapping a single folio is passed back to writeback_iter
		// in the next iteration (or handled on loop exit).
		error = iomap_writepage_map(wpc, wbc, folio);
	// After iterating through all relevant folios, submit any aggregated/pending
	// I/O operations (e.g., ioends containing BIOs) stored in the wpc context.
	// The final error status from the iteration (potentially updated by the last
	// iomap_writepage_map call) is passed to the submission function.
	return iomap_submit_ioend(wpc, error);
}
```