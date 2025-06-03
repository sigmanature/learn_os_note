```C
/**
 * pagecache_isize_extended - update pagecache after extension of i_size
 * @inode:	inode for which i_size was extended
 * @from:	original inode size
 * @to:		new inode size
 *
 * Handle extension of inode size either caused by extending truncate or
 * by write starting after current i_size.  We mark the page straddling
 * current i_size RO so that page_mkwrite() is called on the first
 * write access to the page.  The filesystem will update its per-block
 * information before user writes to the page via mmap after the i_size
 * has been changed.
 *
 * The function must be called after i_size is updated so that page fault
 * coming after we unlock the folio will already see the new i_size.
 * The function must be called while we still hold i_rwsem - this not only
 * makes sure i_size is stable but also that userspace cannot observe new
 * i_size value before we are prepared to store mmap writes at new inode size.
 */
void pagecache_isize_extended(struct inode *inode, loff_t from, loff_t to)
{
	int bsize = i_blocksize(inode); // Filesystem block size
	loff_t rounded_from;
	struct folio *folio;

	WARN_ON(to > inode->i_size); // Sanity check: 'to' should not exceed current i_size

	// If no actual extension or block size is page size or larger, nothing special to do.
	if (from >= to || bsize >= PAGE_SIZE)
		return;

	// Calculate 'from' rounded up to the nearest filesystem block boundary.
	// This helps determine if the page straddling 'from' needs special handling.
	rounded_from = round_up(from, bsize);
	// If 'to' is within this rounded block or if 'rounded_from' is page-aligned,
	// no special handling for the straddling page is needed from this function.
	if (to <= rounded_from || !(rounded_from & (PAGE_SIZE - 1)))
		return;

	// Lock the folio (page) that contains the 'from' offset.
	folio = filemap_lock_folio(inode->i_mapping, from / PAGE_SIZE);
	// If folio is not cached or error, nothing to do.
	if (IS_ERR(folio))
		return;
	/*
	 * See folio_clear_dirty_for_io() for details why folio_mark_dirty()
	 * is needed.
	 * If we make the folio clean (folio_mkclean returns true), we must mark it
	 * dirty again. This ensures that if it was previously dirty, it remains
	 * dirty, and if it was clean, it becomes dirty. This forces a page_mkwrite()
	 * call on subsequent write attempts, allowing the FS to allocate blocks.
	 */
	if (folio_mkclean(folio))
		folio_mark_dirty(folio);

	/*
	 * The post-eof range of the folio must be zeroed before it is exposed
	 * to the file. Writeback normally does this, but since i_size has been
	 * increased we handle it here.
	 */
	// If the folio is (or was just marked) dirty
	if (folio_test_dirty(folio)) {
		unsigned int offset, end;

		// Calculate the range within the folio that corresponds to the extension
		// (from old EOF 'from' to new EOF 'to', capped by folio boundaries).
		offset = from - folio_pos(folio);
		end = min_t(unsigned int, to - folio_pos(folio),
			    folio_size(folio));
		// Zero out this newly extended segment within the folio.
		folio_zero_segment(folio, offset, end);
	}

	folio_unlock(folio); // Unlock the folio
	folio_put(folio);    // Release reference to the folio
}
EXPORT_SYMBOL(pagecache_isize_extended);
```
