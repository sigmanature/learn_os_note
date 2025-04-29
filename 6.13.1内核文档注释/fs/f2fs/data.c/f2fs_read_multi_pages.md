**相关函数和数据结构**
* [decompress_io_ctx](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/f2fs.h/decompress_io_ctx.md)
* [f2fs_mpage_readpages](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/data.c/f2fs_mpage_readpages.md)
```C
// #ifdef CONFIG_F2FS_FS_COMPRESSION // Only if compression is enabled

// Function to read multiple pages belonging to a single compressed cluster
// It finds the compressed blocks, allocates a decompression context (dic),
// issues reads for the compressed blocks, and sets up decompression upon completion.
//压缩上下文总是在栈上分配。就是在f2fs_mpage_readpages中被分配的。
//这个函数很可能在每次一个cluster中的page被填满了才调用一次。注意这些page,也统统来自page cache。
//并且它们的索引对应的统统都是逻辑索引。
int f2fs_read_multi_pages(struct compress_ctx *cc, struct bio **bio_ret,
				unsigned nr_pages, sector_t *last_block_in_bio,
				struct readahead_control *rac, bool for_write)
{
	struct dnode_of_data dn; // Structure to hold direct node information
	struct inode *inode = cc->inode; // Get inode from compress context
	struct f2fs_sb_info *sbi = F2FS_I_SB(inode); // Get F2FS superblock info
	struct bio *bio = *bio_ret; // Get the current bio pointer
	// Calculate the starting page index of the cluster
	unsigned int start_idx = cc->cluster_idx << cc->log_cluster_size;//实锤了cluster_idx必然是文件的逻辑索引。
	sector_t last_block_in_file; // Last valid block index within the file size
	const unsigned int blocksize = F2FS_BLKSIZE; // Filesystem block size (usually 4KB)
	struct decompress_io_ctx *dic = NULL; // Decompression I/O context pointer
	struct extent_info ei = {}; // Structure to hold extent cache information
	bool from_dnode = true; // Flag indicating if block addresses were read from dnode
	int i; // Loop counter
	int ret = 0; // Return value

	// Check for critical filesystem errors (e.g., during checkpoint)
	if (unlikely(f2fs_cp_error(sbi))) {
		ret = -EIO; // Set I/O error
		from_dnode = false; // Don't try to read dnode later
		goto out_put_dnode; // Jump to cleanup
	}

	// Sanity check: ensure the compress context is not empty
	f2fs_bug_on(sbi, f2fs_cluster_is_empty(cc));

	// Calculate the maximum logical block number based on inode size
	last_block_in_file = F2FS_BYTES_TO_BLK(f2fs_readpage_limit(inode) +
							blocksize - 1);

	/* get rid of pages beyond EOF */
	// Iterate through the page slots in the compress context
	for (i = 0; i < cc->cluster_size; i++) {
		struct page *page = cc->rpages[i]; // Get page pointer from context
		struct folio *folio; // Folio pointer

		if (!page) // Skip if no page was requested for this slot
			continue;

		folio = page_folio(page); // Get folio from page
		// Check if the page index is beyond the end of the file
		if ((sector_t)folio->index >= last_block_in_file) {
			// If beyond EOF, zero the page content
			folio_zero_segment(folio, 0, folio_size(folio));
			// Mark it as up-to-date (since it's valid, empty data)
			if (!folio_test_uptodate(folio))
				folio_mark_uptodate(folio);
		} else if (!folio_test_uptodate(folio)) {
			// If page is within file bounds but not already up-to-date, keep it for reading
			continue;
		}
		// If page was beyond EOF or already up-to-date, unlock it
		folio_unlock(folio);
		if (for_write) // If reading for writeback, release the page reference
			folio_put(folio);
		cc->rpages[i] = NULL; // Remove page from context
		cc->nr_rpages--; // Decrement count of pages needed
	} /*整个这个循环干的事情就是遍历在当前cluster中的每个page 只要它的index超出了文件末尾 就对它干以下几件事:
        第一:zero
        第二:不是uptodate的标记为uptodate
        第三:解锁(因为这个函数一般是被f2fs_mpage_readpages调用,处于预读之下的锁保护 所以解锁)
        第四:当前的cc中原始page数组中的槽位清零了
        第五:cc中原始page的数量减去1
        最后剩下的就是那些还在文件范围内的原始数据pages
        问题是如果说是连续的index的话,那肯定是在大阶folio下可以优化的。
    */

	/* we are done since all pages are beyond EOF */
	// If all pages were handled (beyond EOF or already uptodate)
	if (f2fs_cluster_is_empty(cc))
		goto out; // Jump to exit

	// Try to find the compressed block info in the extent cache first
	if (f2fs_lookup_read_extent_cache(inode, start_idx, &ei))
		from_dnode = false; // Found in cache, don't need to read dnode

	if (!from_dnode) // If found in cache or error occurred earlier
		goto skip_reading_dnode; // Skip dnode reading logic

	// Prepare to read the direct node containing the block address for start_idx
	set_new_dnode(&dn, inode, NULL, NULL, 0);
	ret = f2fs_get_dnode_of_data(&dn, start_idx, LOOKUP_NODE);//这个地方是根据簇的起始索引去找到其对应的COMPRESS_ADDR的
	if (ret) // If error getting dnode (e.g., file hole)
		goto out; // Jump to exit

	// For a compressed cluster, the first address slot should contain COMPRESS_ADDR
	f2fs_bug_on(sbi, dn.data_blkaddr != COMPRESS_ADDR);

skip_reading_dnode: // Label to jump to if using extent cache or after reading dnode
	// Determine the number of compressed pages (nr_cpages)
	// The actual block addresses start from the *second* slot in the dnode entry
	// or from ei.blk in the extent cache.
	// 这一步干的事就是从起始的cluster索引一直到cluster里所有的page找到对应的数据块。
	// 最终为压缩上下文保存所有找到的数据块。
	cc->nr_cpages = 0; // Initialize count of compressed blocks
	for (i = 1; i < cc->cluster_size; i++) { /* Check subsequent slots/blocks 注意到i从1开始
        因为啊i=0这个对应的dn.ofs_in_node 也就是我们根据刚刚的start_bidx查找到的那个块啊
        一定被预留为了COMPRESS_ADDR*/
		block_t blkaddr; // Physical block address of a compressed chunk

		// Get block address either from dnode (offset +i) or extent cache (ei.blk + i - 1)
		blkaddr = from_dnode ? data_blkaddr(dn.inode, dn.node_folio,
					dn.ofs_in_node + i) :
					ei.blk + i - 1; /* Note: ei.blk is the first *data* block addr 根据extent查到的
                    竟然直接对应的就是第一个数据块地址了?*/

		// Stop if we encounter an invalid address (marks end of compressed blocks)
		if (!__is_valid_data_blkaddr(blkaddr))
			break;

		// Check if the block address is valid within the filesystem bounds
		if (!f2fs_is_valid_blkaddr(sbi, blkaddr, DATA_GENERIC)) {
			ret = -EFAULT; // Invalid address error
			goto out_put_dnode; // Jump to cleanup
		}
		cc->nr_cpages++; // Increment count of compressed blocks found

		// If using extent cache, stop when we reach the cached length
		if (!from_dnode && i >= ei.c_len) // ei.c_len is the number of compressed blocks
			break;
	}

	/* nothing to decompress */
	// If no valid compressed block addresses were found (shouldn't happen if COMPRESS_ADDR was set)
	if (cc->nr_cpages == 0) {
		ret = 0; // No error, but nothing to do
		goto out_put_dnode; // Jump to cleanup
	}

	// Allocate the Decompression I/O Context 
	dic = f2fs_alloc_dic(cc);
    /*分配解压缩上下文的时候,会将压缩上下文的所有原始数据page拷贝过来*/
	if (IS_ERR(dic)) { // Check for allocation errors
		ret = PTR_ERR(dic);
		goto out_put_dnode; // Jump to cleanup
	}

	// Loop through the compressed blocks that need to be read
	for (i = 0; i < cc->nr_cpages; i++) {
		// Get the folio corresponding to the i-th compressed page buffer allocated in f2fs_alloc_dic
		struct folio *folio = page_folio(dic->cpages[i]);
		block_t blkaddr; // Physical block address
		struct bio_post_read_ctx *ctx; // Context for bio completion handler
		// Get the physical block address for the i-th compressed block
		// Note the offset: + i + 1 for dnode, + i for extent cache
		blkaddr = from_dnode ? data_blkaddr(dn.inode, dn.node_folio,
					dn.ofs_in_node + i + 1) :
					ei.blk + i;

		// Ensure any previous writes to this block are completed
		f2fs_wait_on_block_writeback(inode, blkaddr);/*这些被压缩的数据块全部视作元数据,读到的是compress_mapping之中*/

		// Check if this compressed block is already cached (optimization)
		if (f2fs_load_compressed_folio(sbi, folio, blkaddr)) {
			// If cached, decrement remaining count. If it was the last one, trigger decompression.
			if (atomic_dec_and_test(&dic->remaining_pages)) {
				f2fs_decompress_cluster(dic, true); // Decompress now
				break; // Exit loop, all data available
			}
			continue; // Go to next compressed block
		}

		// Check if the current bio exists and if the new read can be merged
		if (bio && (!page_is_mergeable(sbi, bio, // Check physical contiguity/size limits
					*last_block_in_bio, blkaddr) ||
		    // Check if crypto contexts allow merging (if encryption is enabled)
		    !f2fs_crypt_mergeable_bio(bio, inode, folio->index, NULL))) {
submit_and_realloc: // Label for submitting existing bio and starting a new one
			f2fs_submit_read_bio(sbi, bio, DATA); // Submit the current bio
			bio = NULL; // Reset bio pointer
		}

		// If no current bio exists (or previous one was just submitted)
		if (!bio) {
			// Allocate a new bio for reading
			// nr_pages is a hint for bio size; f2fs_ra_op_flags sets read-ahead flags
			// folio->index here is the index of the *compressed page buffer*, used for context
			bio = f2fs_grab_read_bio(inode, blkaddr, nr_pages,
					f2fs_ra_op_flags(rac),
					folio->index, for_write);
                    /*提一嘴f2fs_grab_read_bio分配bio的方式。你可以注意到这初次的分配直接分配了
                    整个所有bio大小*/
                    /**/
			if (IS_ERR(bio)) { // Handle bio allocation error
				ret = PTR_ERR(bio);
				// Signal error to decompression context and clean it up
				f2fs_decompress_end_io(dic, ret, true);
				f2fs_put_dnode(&dn); // Release dnode if held
				*bio_ret = NULL; // Ensure caller doesn't use the failed bio
				return ret; // Return error
			}
		}

		// Add the compressed page buffer (folio) to the bio
		// This tells the block layer to read 'blocksize' bytes from 'blkaddr' into this folio's page
		if (!bio_add_folio(bio, folio, blocksize, 0))
			goto submit_and_realloc; // If bio is full, submit and retry adding

		// Get the post-read context attached to the bio
		ctx = get_post_read_ctx(bio);
		// Enable the decompression step in the bio completion handler
		ctx->enabled_steps |= STEP_DECOMPRESS;
		// Increment the reference count of the decompression context (dic)
		// The bio completion handler will decrement it later.
		refcount_inc(&dic->refcnt);

		// Update statistics
		inc_page_count(sbi, F2FS_RD_DATA);
		f2fs_update_iostat(sbi, inode, FS_DATA_READ_IO, F2FS_BLKSIZE);
		// Update the last block added to this bio for merge checking
		*last_block_in_bio = blkaddr;
	}

	// Release the dnode reference if it was acquired
	if (from_dnode)
		f2fs_put_dnode(&dn);

	// Update the caller's bio pointer with the potentially new/modified bio
	*bio_ret = bio;
	return 0; // Success

out_put_dnode: // Cleanup path if dnode was potentially acquired
	if (from_dnode)
		f2fs_put_dnode(&dn);
out: // General exit/error path
	// If an error occurred before submitting reads, mark target pages as !Uptodate and unlock
	for (i = 0; i < cc->cluster_size; i++) {
		if (cc->rpages[i]) {
			ClearPageUptodate(cc->rpages[i]);
			unlock_page(cc->rpages[i]);
		}
	}
	// Update the caller's bio pointer
	*bio_ret = bio;
	return ret; // Return error code or 0 if exiting cleanly
}
```
