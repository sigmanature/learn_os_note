**相关函数**
* [folio_fill_tail](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/include/linux/highmem.h/folio_fill_tail.md)
```C
static int iomap_read_inline_data(const struct iomap_iter *iter,
		struct folio *folio)
{
	const struct iomap *iomap = iomap_iter_srcmap(iter);
	/* Calculate the size of inline data to copy.
	 * It's the file size minus the offset of the inline extent, which should be the remaining part of the file.
	 */
	size_t size = i_size_read(iter->inode) - iomap->offset;/*实际上这个才应该是真正的inline的文件剩余的长度*/
	/* Calculate the offset within the folio where inline data should be copied.
	 * offset_in_folio calculates the offset of iomap->offset within the folio's address range.
	 */
	size_t offset = offset_in_folio(folio, iomap->offset);

	/* Check if the folio is already uptodate.
	 * If it is, no need to read again, return success.
	 */
	if (folio_test_uptodate(folio))
		return 0;

	/* Warn if the calculated inline data size is greater than the extent length.
	 * This should not happen if the iomap is correctly constructed, so it's a warning for potential errors.
	 */
	if (WARN_ON_ONCE(size > iomap->length))
		return -EIO; /* Return I/O error if size is inconsistent. */
    /*正常情况下只有iomap.length超过size的份 这是要考虑的边界情况*/
	/* Check if the offset within the folio is non-zero.
	 * If offset > 0, it means inline data is not starting from the beginning of the folio.
	 */
	if (offset > 0)
		/* Allocate internal resources if needed when offset is not zero.
		 * ifs_alloc might be filesystem-specific allocation for internal structures,
		 * potentially related to handling non-page-aligned inline data.
		 */
		ifs_alloc(iter->inode, folio, iter->flags); /* 'iter->flags' might control allocation behavior. */
        /*天杀的 还是gfp flag*/

	/* Fill the folio with inline data and pad the rest with zeroes.
	 * folio_fill_tail copies 'size' bytes from 'iomap->inline_data' to 'folio' starting at 'offset',
	 * and then zeroes out the remaining part of the folio.
	 */
	folio_fill_tail(folio, offset, iomap->inline_data, size);
	/* Mark the range of the folio that now contains valid data as uptodate.
	 * iomap_set_range_uptodate marks the range from 'offset' to the end of the folio as uptodate.
	 */
	iomap_set_range_uptodate(folio, offset, folio_size(folio) - offset);
	/* Return 0 to indicate successful inline data read. */
	return 0;
}
```