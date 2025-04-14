```C
/**
 * folio_zero_tail - Zero the tail of a folio.
 * @folio: The folio to zero.
 * @offset: The byte offset in the folio to start zeroing at.
 * @kaddr: The address the folio is currently mapped to.
 *
 * If you have already used kmap_local_folio() to map a folio, written
 * some data to it and now need to zero the end of the folio (and flush
 * the dcache), you can use this function.  If you do not have the
 * folio kmapped (eg the folio has been partially populated by DMA),
 * use folio_zero_range() or folio_zero_segment() instead.
 *
 * Return: An address which can be passed to kunmap_local().
 */
static inline __must_check void *folio_zero_tail(struct folio *folio,
		size_t offset, void *kaddr)
{
	size_t len = folio_size(folio) - offset;

	/* Handle HIGHMEM folios in chunks. */
	if (folio_test_highmem(folio)) {
		size_t max = PAGE_SIZE - offset_in_page(offset);

		/* Loop to zero in page-sized chunks if needed in HIGHMEM. */
		while (len > max) {
			/* Zero 'max' bytes. */
			memset(kaddr, 0, max);
			/* Unmap the current portion. */
			kunmap_local(kaddr);
			/* Update remaining length and offset. */
			len -= max;
			offset += max;
			/* Reset max to PAGE_SIZE for subsequent chunks. */
			max = PAGE_SIZE;
			/* Map the next portion of the folio. */
			kaddr = kmap_local_folio(folio, offset);
		}
	}

	/* Zero the remaining tail (less than or equal to PAGE_SIZE). */
	memset(kaddr, 0, len);
	/* Flush the data cache for the folio to ensure zeroing is written to memory. */
	flush_dcache_folio(folio);

	/* Return the current kernel address, which can be used for kunmap_local later if needed. */
	return kaddr;
}
```