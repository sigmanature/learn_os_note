```C
/* Context for decompressing one cluster on the read IO path */
struct decompress_io_ctx {
	u32 magic;			/* magic number to indicate page is compressed */
	struct inode *inode;		/* inode the context belong to */
	pgoff_t cluster_idx;		/* cluster index number */
	unsigned int cluster_size;	/* page count in cluster */
	unsigned int log_cluster_size;	/* log of cluster size */
	struct page **rpages;		/* pages store raw data in cluster */
	unsigned int nr_rpages;		/* total page number in rpages */
	struct page **cpages;		/* pages store compressed data in cluster */
	unsigned int nr_cpages;		/* total page number in cpages */
	struct page **tpages;		/* temp pages to pad holes in cluster */
	void *rbuf;			/* virtual mapped address on rpages */
	struct compress_data *cbuf;	/* virtual mapped address on cpages */
	size_t rlen;			/* valid data length in rbuf */
	size_t clen;			/* valid data length in cbuf */

	/*
	 * The number of compressed pages remaining to be read in this cluster.
	 * This is initially nr_cpages.  It is decremented by 1 each time a page
	 * has been read (or failed to be read).  When it reaches 0, the cluster
	 * is decompressed (or an error is reported).
	 *
	 * If an error occurs before all the pages have been submitted for I/O,
	 * then this will never reach 0.  In this case the I/O submitter is
	 * responsible for calling f2fs_decompress_end_io() instead.
	 */
	atomic_t remaining_pages;

	/*
	 * Number of references to this decompress_io_ctx.
	 *
	 * One reference is held for I/O completion.  This reference is dropped
	 * after the pagecache pages are updated and unlocked -- either after
	 * decompression (and verity if enabled), or after an error.
	 *
	 * In addition, each compressed page holds a reference while it is in a
	 * bio.  These references are necessary prevent compressed pages from
	 * being freed while they are still in a bio.
	 */
	refcount_t refcnt;

	bool failed;			/* IO error occurred before decompression? */
	bool need_verity;		/* need fs-verity verification after decompression? */
	void *private;			/* payload buffer for specified decompression algorithm */
	void *private2;			/* extra payload buffer */
	struct work_struct verity_work;	/* work to verify the decompressed pages */
	struct work_struct free_work;	/* work for late free this structure itself */
};
```