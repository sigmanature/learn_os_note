```C
/*
 * iomap_writepage_handle_eof - Check and handle folio interaction with EOF.
 * @folio: The folio being written.
 * @inode: The corresponding inode.
 * @end_pos: In/Out pointer to the folio's end position relative to file start.
 *           Will be adjusted downwards if the folio straddles EOF.
 *
 * Checks if the folio lies partially or fully beyond the inode's size (i_size).
 * If it's fully beyond, returns false.
 * If it straddles i_size, zeros the portion beyond i_size within the folio's
 * buffer and updates @end_pos to match i_size.
 *
 * Return: true if the folio is (at least partially) within i_size and should
 *         be processed further, false otherwise.
 */
static bool iomap_writepage_handle_eof(struct folio *folio, struct inode *inode,
		u64 *end_pos)
{
	// Read the current size of the file.
	u64 isize = i_size_read(inode);

	// Check if the folio's calculated end position extends beyond the file size.
	if (*end_pos > isize) {
		// Calculate the offset within the folio where the file actually ends.
		size_t poff = offset_in_folio(folio, isize);
		// Calculate the page index corresponding to the last page containing file data.
		pgoff_t end_index = isize >> PAGE_SHIFT;

		/*
		 * Check if the folio is entirely outside of i_size.
		 */
		// Case 1: The folio's starting index is already past the end index.
		// Case 2: The folio's index *is* the end index, but the file ends
		// exactly at the beginning of this folio (poff == 0).
		// Includes check for index overflow on 32-bit systems as described in comments.
		if (folio->index > end_index ||
		    (folio->index == end_index && poff == 0))
			// Folio is entirely beyond EOF, skip it.
			return false;

		/*
		 * The folio straddles i_size. Zero the unused part.
		 * This is necessary because the memory might be mmapped, and the
		 * area beyond EOF must appear as zeros to userspace.
		 */
		folio_zero_segment(folio, poff, folio_size(folio));
		// Adjust the effective end position for writeback to the actual EOF.
		*end_pos = isize;
	}

	// Return true, indicating the folio contains data within i_size and needs processing.
	return true;
}
```