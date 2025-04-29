```C
// Allocate and initialize a Decompression I/O Context (dic)
struct decompress_io_ctx *f2fs_alloc_dic(struct compress_ctx *cc)
{
	struct decompress_io_ctx *dic; // Pointer to the context structure
	// Calculate the starting page index of the cluster
	pgoff_t start_idx = start_idx_of_cluster(cc);/*cc->cluster_idx<<cc->log_cluster_size*/
	struct f2fs_sb_info *sbi = F2FS_I_SB(cc->inode); // Filesystem superblock info
	int i, ret; // Loop counter, return value

	// Allocate the main dic structure from a slab cache dic已经在内存中了
	dic = f2fs_kmem_cache_alloc(dic_entry_slab, GFP_F2FS_ZERO, false, sbi);
	if (!dic)
		return ERR_PTR(-ENOMEM); // Return error if allocation fails

	// Allocate an array to hold pointers to the *target* pages (where decompressed data will go)
	dic->rpages = page_array_alloc(cc->inode, cc->cluster_size);
	if (!dic->rpages) {
		kmem_cache_free(dic_entry_slab, dic); // Free the dic structure
		return ERR_PTR(-ENOMEM); // Return error
	}

	// Initialize dic fields
	dic->magic = F2FS_COMPRESSED_PAGE_MAGIC; // Magic number for identification
	dic->inode = cc->inode; // Store inode pointer
	// Set the number of compressed pages we expect to read (determined in f2fs_read_multi_pages)
	atomic_set(&dic->remaining_pages, cc->nr_cpages);
	// Copy cluster information from the compress context (cc)
	dic->cluster_idx = cc->cluster_idx;
	dic->cluster_size = cc->cluster_size;
	dic->log_cluster_size = cc->log_cluster_size;
	dic->nr_cpages = cc->nr_cpages; // Store the number of compressed pages
	refcount_set(&dic->refcnt, 1); // Initialize reference count to 1 (for this allocation)
	dic->failed = false; // Initialize error flag
	// Check if filesystem-level verification (fs-verity) is needed for this cluster
	dic->need_verity = f2fs_need_verity(cc->inode, start_idx);

	// Copy the target page pointers from the compress context (cc) to the decompress context (dic)
	// These are the pages the original read request wanted.
	for (i = 0; i < dic->cluster_size; i++)
		dic->rpages[i] = cc->rpages[i]; /*把所有的rpage全拷贝进来*/
	dic->nr_rpages = cc->cluster_size; // Store the total number of slots in rpages (cluster size)

	// Allocate an array to hold pointers to the *compressed* pages/buffers
	// These pages will temporarily store the data read from disk before decompression.
	dic->cpages = page_array_alloc(dic->inode, dic->nr_cpages);
	if (!dic->cpages) {
		ret = -ENOMEM;
		goto out_free; // Jump to cleanup on error
	}

	// Allocate actual page structures for the compressed data buffers
	for (i = 0; i < dic->nr_cpages; i++) {
		struct page *page;

		// Allocate a page (likely from a special pool for compression buffers)
		page = f2fs_compress_alloc_page();/*这些page也没有对应的mapping 你只能说它们就是一大堆缓冲区*/
		// Set metadata on the page linking it back to the dic (used in completion handlers)
        /*这个page/folio的private会指向其解压缩上下文*/
		f2fs_set_compressed_page(page, cc->inode,
					start_idx + i + 1, dic); /*
                    很奇怪 为什么compressed page的逻辑索引要多一个
                    dic 成为了这些compress page的私有数据
                    */
		dic->cpages[i] = page; // Store the allocated page pointer
	}

	// Prepare memory buffers (rbuf, cbuf) needed for decompression
	ret = f2fs_prepare_decomp_mem(dic, true);
	if (ret)
		goto out_free; // Jump to cleanup on error

	return dic; // Return the initialized decompression context

out_free: // Error cleanup path
	f2fs_free_dic(dic, true); // Free all allocated resources for dic
	return ERR_PTR(ret); // Return error code
}
```