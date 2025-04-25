```C
/* compress context */
struct compress_ctx {
	struct inode *inode;		/* inode the context belong to */
	pgoff_t cluster_idx;		/* cluster index number */
	unsigned int cluster_size;	/* page count in cluster */
	unsigned int log_cluster_size;	/* log of cluster size */
	struct page **rpages;		/* pages store raw data in cluster 核心的数据结构 page数组啊*/
	unsigned int nr_rpages;		/* total page number in rpages */
	struct page **cpages;		/* pages store compressed data in cluster */
	unsigned int nr_cpages;		/* total page number in cpages */
	unsigned int valid_nr_cpages;	/* valid page number in cpages */
	void *rbuf;			/* virtual mapped address on rpages */
	struct compress_data *cbuf;	/* virtual mapped address on cpages */
	size_t rlen;			/* valid data length in rbuf 注意这个valid 用词相当微妙可以说*/
	size_t clen;			/* valid data length in cbuf 注意这个valid 用词相当微妙可以说*/
	void *private;			/* payload buffer for specified compression algorithm 我们已经看到lz4就使用了这个成员了*/
	void *private2;			/* extra payload buffer */
};
// Calculate the offset of a page index within its cluster
static unsigned int offset_in_cluster(struct compress_ctx *cc, pgoff_t index)
{
	// index: logical page index within the file
	// cc->cluster_size: number of pages per cluster (power of 2)
	// Uses bitwise AND as a fast modulo operation for powers of 2
	// (cluster_size - 1) creates a bitmask (e.g., if cluster_size is 4, mask is 0b11)
	return index & (cc->cluster_size - 1);
}

// Calculate the cluster index for a given page index
static pgoff_t cluster_idx(struct compress_ctx *cc, pgoff_t index)
{
	// index: logical page index within the file
	// cc->log_cluster_size: log2 of the number of pages per cluster
	// Right-shifting by log_cluster_size is equivalent to dividing by cluster_size
	return index >> cc->log_cluster_size;
}

// Calculate the starting page index of the current cluster in the context
static pgoff_t start_idx_of_cluster(struct compress_ctx *cc)
{
	// cc->cluster_idx: the index of the current cluster being processed
	// cc->log_cluster_size: log2 of the number of pages per cluster
	// Left-shifting by log_cluster_size is equivalent to multiplying by cluster_size
	return cc->cluster_idx << cc->log_cluster_size;
}
static void *page_array_alloc(struct inode *inode, int nr)
{
	struct f2fs_sb_info *sbi = F2FS_I_SB(inode);
	unsigned int size = sizeof(struct page *) * nr;

	if (likely(size <= sbi->page_array_slab_size))
		return f2fs_kmem_cache_alloc(sbi->page_array_slab,
					GFP_F2FS_ZERO, false, F2FS_I_SB(inode));
	return f2fs_kzalloc(sbi, size, GFP_NOFS);
}

static void page_array_free(struct inode *inode, void *pages, int nr)
{
	struct f2fs_sb_info *sbi = F2FS_I_SB(inode);
	unsigned int size = sizeof(struct page *) * nr;

	if (!pages)
		return;

	if (likely(size <= sbi->page_array_slab_size))
		kmem_cache_free(sbi->page_array_slab, pages);
	else
		kfree(pages);
}

// Add a page (folio) to the compression context
void f2fs_compress_ctx_add_page(struct compress_ctx *cc, struct folio *folio)
{
	unsigned int cluster_ofs; // Offset of the page within its cluster

	// Sanity check: Ensure the page actually belongs to the cluster currently in the context
	if (!f2fs_cluster_can_merge_page(cc, folio->index))
		// If not, trigger a bug/warning (should have been handled in f2fs_mpage_readpages)
		f2fs_bug_on(F2FS_I_SB(cc->inode), 1);

	// Calculate the offset (0 to cluster_size-1) for this page within the cluster
	cluster_ofs = offset_in_cluster(cc, folio->index);
	// Store the page pointer (folio_page gets the 'struct page *' from 'struct folio *')
	// into the rpages array at the calculated offset.
	// 'rpages' acts as a temporary holder for pages belonging to the cluster being processed.
	cc->rpages[cluster_ofs] = folio_page(folio, 0);
	// Increment the count of pages currently held in the context for this cluster
	cc->nr_rpages++;
	// Set/update the cluster index this context is currently handling
	cc->cluster_idx = cluster_idx(cc, folio->index);
}

// Function to destroy the generic part of the compression context
void f2fs_destroy_compress_ctx(struct compress_ctx *cc, bool reuse)
{
	// Free the array holding pointers to the original pages (rpages)
	// page_array_free handles freeing the array itself, not the pages within it
	page_array_free(cc->inode, cc->rpages, cc->cluster_size);
	cc->rpages = NULL; // Set pointer to NULL
	cc->nr_rpages = 0; // Reset count of original pages
	cc->nr_cpages = 0; // Reset count of compressed pages/blocks (likely related to write)
	cc->valid_nr_cpages = 0; // Reset count of valid compressed pages/blocks (likely related to write)
	if (!reuse) // If the context is not going to be immediately reused for the same cluster
		// Reset the cluster index to indicate no cluster is currently active
		cc->cluster_idx = NULL_CLUSTER;
}
```