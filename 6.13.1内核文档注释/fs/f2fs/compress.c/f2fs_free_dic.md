```C
// Free resources associated with a decompress_io_ctx
static void f2fs_free_dic(struct decompress_io_ctx *dic,
		bool bypass_destroy_callback)
{
	int i; // Loop counter

	// Release mapped memory buffers (rbuf, cbuf) and algorithm-specific context
	f2fs_release_decomp_mem(dic, bypass_destroy_callback, true);

	// Free temporary pages allocated in f2fs_prepare_decomp_mem (tpages)
	if (dic->tpages) {
		for (i = 0; i < dic->cluster_size; i++) {
			// Skip if the slot pointed to an original requested page (rpage)
			if (dic->rpages[i])
				continue;
			// Skip if no temporary page was allocated for this slot
			if (!dic->tpages[i])
				continue;
			// Free the temporary page
			f2fs_compress_free_page(dic->tpages[i]);
		}
		// Free the tpages array itself
		page_array_free(dic->inode, dic->tpages, dic->cluster_size);
	}

	// Free the compressed page buffers (cpages)
	if (dic->cpages) {
		for (i = 0; i < dic->nr_cpages; i++) {
			if (!dic->cpages[i]) // Skip if slot is empty
				continue;
			// Free the page used for the compressed buffer
			f2fs_compress_free_page(dic->cpages[i]);
		}
		// Free the cpages array itself
		page_array_free(dic->inode, dic->cpages, dic->nr_cpages);
	}

	// Free the rpages array (just the array, not the pages it pointed to)
	page_array_free(dic->inode, dic->rpages, dic->nr_rpages);
	// Free the main dic structure itself from the slab cache
	kmem_cache_free(dic_entry_slab, dic);
}

```