```C
/**
 * struct file_ra_state - Track a file's readahead state.
 * @start: Where the most recent readahead started.
 * @size: Number of pages read in the most recent readahead.
 * @async_size: Numer of pages that were/are not needed immediately
 *      and so were/are genuinely "ahead".  Start next readahead when
 *      the first of these pages is accessed.
 * @ra_pages: Maximum size of a readahead request, copied from the bdi.
 * @mmap_miss: How many mmap accesses missed in the page cache.
 * @prev_pos: The last byte in the most recent read request.
 *
 * When this structure is passed to ->readahead(), the "most recent"
 * readahead means the current readahead.
 */
struct file_ra_state {
	pgoff_t start;
	unsigned int size;
	unsigned int async_size;
	unsigned int ra_pages;
	unsigned int mmap_miss;
	loff_t prev_pos;
};

```