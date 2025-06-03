```C
static int f2fs_write_single_folio(struct folio *folio, int *submitted_pages_count,
				struct bio **bio_ptr, sector_t *last_block_ptr,
				struct writeback_control *wbc,
				enum iostat_type io_type,
				unsigned int compr_blocks, bool is_reclaim,
				u64 dirty_pos_start, u64 dirty_pos_end)
{
	struct inode *inode = folio->mapping->host;
	struct f2fs_sb_info *sbi = F2FS_I_SB(inode);
	struct f2fs_iomap_folio_state *ifs = folio->private; // ifs should be set by caller if folio_nr_pages > 1
	pgoff_t folio_start_index = folio->index;
	int err = 0;
	int local_submitted_count = 0;

	// Calculate the page indices within the folio that correspond to the dirty range
	// The dirty_pos_start/end are file offsets. We need folio-relative page indices.
	pgoff_t first_page_idx_in_folio = (dirty_pos_start - folio_pos(folio)) >> PAGE_SHIFT;
	pgoff_t last_page_idx_in_folio = (dirty_pos_end - 1 - folio_pos(folio)) >> PAGE_SHIFT;

	for (pgoff_t i = first_page_idx_in_folio; i <= last_page_idx_in_folio; ++i) {
		struct page *page = folio_page(folio, i);
		pgoff_t current_page_file_index = folio_start_index + i;

		// Skip if page is beyond EOF (i_size_read is expensive here,
		// writeback_iter and iomap_writepage_handle_eof should have handled major EOF issues.
		// However, a check against wbc->range_end or a precise end_index might be useful
		// if not fully covered by iomap_find_dirty_range logic)
		// For large folios, increment pending bytes before attempting to write the page
		if (ifs) {
			atomic_add(PAGE_SIZE, &ifs->write_bytes_pending);
		}

		struct f2fs_io_info fio = {
			.sbi = sbi,
			.ino = inode->i_ino,
			.type = DATA, // Assuming DATA type, adjust if node pages are handled here
			.op = REQ_OP_WRITE,
			.op_flags = wbc_to_write_flags(wbc),
			.old_blkaddr = NULL_ADDR,
			.page = page,
			.encrypted_page = NULL,
			.submitted = 0, // f2fs_do_write_data_page will update this if it submits
			.compr_blocks = compr_blocks, // Pass this through
			.need_lock = compr_blocks ? LOCK_DONE : LOCK_RETRY,
			.meta_gc = f2fs_meta_inode_gc_required(inode) ? 1 : 0,
			.io_type = io_type,
			.io_wbc = wbc,
			// For bio and last_block, F2FS has its own merging.
			// If f2fs_do_write_data_page uses them directly, pass pointers.
			// If it's about merging *across calls* to f2fs_do_write_data_page,
			// that logic is more complex and currently sits higher up or in IPU.
			// For simplicity here, let's assume f2fs_do_write_data_page
			// handles its bio needs internally or via fio->bio.
			// The bio_ptr and last_block_ptr are from the original f2fs_write_single_data_page
			// which might have tried to merge. Here, each page is distinct for fio.
			.bio = bio_ptr ? *bio_ptr : NULL, // Allow passing bio for potential internal use
			.last_block = last_block_ptr ? *last_block_ptr : 0,
		};

		// Retain original retry logic for f2fs_do_write_data_page
		err = f2fs_do_write_data_page(&fio);
		if (err == -EAGAIN) {
			f2fs_bug_on(sbi, compr_blocks && err == -EAGAIN); // Original bug_on
			fio.need_lock = LOCK_REQ;
			err = f2fs_do_write_data_page(&fio);
		}

		if (bio_ptr) // Update the caller's bio if f2fs_do_write_data_page modified it
			*bio_ptr = fio.bio;
		if (last_block_ptr)
			*last_block_ptr = fio.last_block;


		if (err) {
			// If this page failed to be prepared for write (e.g. -ENOSPC),
			// decrement pending_bytes if we incremented it.
			if (ifs) {
				// If submission failed for this page, it won't go to f2fs_write_end_io
				// for this portion.
				if (atomic_sub_and_test(PAGE_SIZE, &ifs->write_bytes_pending)) {
					// This implies all other pending IO for this folio also completed/failed
					// and this was the last one.
                                        // This scenario is tricky: if other pages *were* submitted,
                                        // they will call folio_end_writeback. If this is the *only*
                                        // page and it fails, then folio_end_writeback needs to be called
                                        // by the caller (f2fs_write_cache_folios) after unlock.
                                        // The iomap model handles this by a bias count, which is
                                        // decremented after the loop. If it hits 0, and no IO was submitted,
                                        // then it calls folio_end_writeback.
                                        // For now, just decrement. The caller will handle overall folio_end_writeback.
				}
			}
			// An error on one page within the folio typically means we stop
			// processing this folio and report the error.
			// The VFS writeback loop (writeback_iter) might then requeue the folio.
			mapping_set_error(folio->mapping, err); // Report error to mapping
			break; // Stop processing this folio on error
		} else {
			// Page successfully processed by f2fs_do_write_data_page
			// fio.submitted should reflect if f2fs_do_write_data_page actually
			// queued something (it's usually 1 for a single page, or more if it did internal merging).
			// The original f2fs_write_single_data_page updated *submitted_count.
			// Here, we count pages for which f2fs_do_write_data_page succeeded.
			local_submitted_count++;
			if (wbc->nr_to_write > 0) // Only decrement if wbc cares
				wbc->nr_to_write--; // Decrement for each page successfully handled
		}

		// If wbc->nr_to_write is a budget and we've exhausted it.
		if (wbc->nr_to_write == 0 && wbc->sync_mode == WB_SYNC_NONE) {
			err = -EAGAIN; // Signal to stop, budget met
			break;
		}
	}

	if (submitted_pages_count)
		*submitted_pages_count = local_submitted_count;

	return err;
}
```C

```C
static int f2fs_write_cache_folios(struct address_space *mapping,
				   struct writeback_control *wbc,
				   enum iostat_type io_type)
{
	struct folio *folio =
		NULL; // Renamed from 'folio' to avoid conflict if needed
	// bio and last_block are typically managed per-fio or by submission functions in F2FS,
	// not usually at this top level directly for merging individual pages.
	// struct bio *bio = NULL;
	// sector_t last_block = 0;
	int err = 0;
	struct inode *inode = mapping->host;
	bool is_compressed_file = f2fs_compressed_file(inode);
	struct compress_ctx cc;
	int iter_err = 0; // Error from writeback_iter
#ifdef CONFIG_F2FS_FS_COMPRESSION
	if (is_compressed_file) {
		// Initialize compress_ctx structure fields (inode, sizes, etc.)
		cc.inode = inode;
		cc.log_cluster_size = F2FS_I(inode)->i_log_cluster_size;
		cc.cluster_size = 1 << cc.log_cluster_size;
		// cc.rpages and cc.cpages will be allocated by f2fs_init_compress_ctx
		cc.rpages = NULL;
		cc.cpages = NULL;
		cc.cluster_idx = NULL_CLUSTER;
		cc.nr_rpages = 0;
		cc.nr_cpages = 0;
		cc.valid_nr_cpages = 0;
		if (f2fs_init_compress_ctx(&cc)) {
			is_compressed_file = false;
		}
	}
#endif
	if (get_dirty_pages(mapping->host) <=
	    SM_I(F2FS_M_SB(mapping))->min_hot_blocks)
		set_inode_flag(mapping->host, FI_HOT_DATA);
	else
		clear_inode_flag(mapping->host, FI_HOT_DATA);

	while ((folio = writeback_iter(mapping, wbc, folio, &iter_err))) {
		struct f2fs_iomap_folio_state *fifs = NULL;
		u64 pos = folio_pos(folio);
		u64 end_pos = pos + folio_size(folio);
		u64 end_pos_aligned;
		u32 r_len;
		int submitted_pages_this_folio = 0;
		if (!iomap_writepage_handle_eof(folio, inode, &end_pos)) {
			folio_unlock(folio);
			return 0;
		}

		fifs = folio->private;
		if (i_blocks_per_folio(inode, folio) > 1) {
			if (!fifs) {
				fifs = fifs_alloc(inode, folio, 0);
				iomap_set_range_dirty(folio, 0, end_pos - pos);
			}

			/*
		 * Keep the I/O completion handler from clearing the writeback
		 * bit until we have submitted all blocks by adding a bias to
		 * fifs->write_bytes_pending, which is dropped after submitting
		 * all blocks.
		 */
			WARN_ON_ONCE(atomic_read(&fifs->write_bytes_pending) !=
				     0);
			atomic_inc(&fifs->write_bytes_pending);
		}
		folio_start_writeback(folio);
		
		end_aligned = round_up(end_pos, i_blocksize(inode));
		while ((r_len = iomap_find_dirty_range(folio, &pos,
						       end_aligned))) {
			if (is_compressed_file) {
				pgoff_t cluster_idx =
					cluster_idx(&cc, pos >> PAGE_SHIFT);

				// If current dirty range belongs to a new cluster than what's in cc
				if (cc.cluster_idx != NULL_CLUSTER &&
				    cc.cluster_idx != cluster_idx) {
					int submitted_compress = 0;
					if (!f2fs_cluster_is_empty(&cc)) {
						err = f2fs_write_multi_pages(
							&cc,
							&submitted_compress,
							wbc, io_type);
						// wbc->nr_to_write -= submitted_compress;
						/* writeback_iter choose to dec folio_nr_pages in wbc->nr_to_write
						 * completly different from the dec policy that f2fs choose in f2fs_write_cache_pages
						 * it's an open problem here
						*/
						if (err) {
							if (err == -EAGAIN)
								wbc->nr_to_write =
									0; // Mark budget met
							goto handle_current_folio_error;
						}
					}
					f2fs_destroy_compress_ctx(
						&cc,
						false); // false to reuse buffers, resets nr_rpages, keeps cluster_idx NULL
					f2fs_init_compress_ctx(
						&cc); // Ensure rpages is valid
					cc.cluster_idx =
						NULL_CLUSTER; // Explicitly ensure for next logic
				}

				// If cc is now for a new/empty cluster, prepare it
				if (f2fs_cluster_is_empty(&cc)) {
					cc.cluster_idx =
						cluster_idx; // Set for prepare_compress_overwrite
					int prep_ret = prepare_compress_overwrite(
						&cc, NULL,
						pos >>
							PAGE_SHIFT, /* pgoff_t index */
						NULL);
					if (prep_ret < 0) {
						err = prep_ret;
						goto handle_current_folio_error;
					}
					// After prepare_compress_overwrite, cc.cluster_idx is definitively set,
					// and cc.rpages might be filled if it was an overwrite.
				}

				// Add the current dirty range from folio to cc
				// f2fs_compress_ctx_add_folio does not do folio_get.
				// The folio is locked. cc needs a ref.
				folio_get(folio); // cc takes a reference
				f2fs_compress_ctx_add_folio(&cc, folio,pos,r_len);
				
				// The ref taken by folio_get will be put by f2fs_put_rpages_wbc (via f2fs_write_multi_pages)

				if (f2fs_cluster_is_full(&cc)) {
					int submitted_compress = 0;
					err = f2fs_write_multi_pages(
						&cc, &submitted_compress, wbc,
						io_type);
					// wbc->nr_to_write -= submitted_compress;
					if (err) {
						if (err == -EAGAIN)
							wbc->nr_to_write = 0;
						goto handle_current_folio_error;
					}
					f2fs_destroy_compress_ctx(
						&cc,
						false); // Reset for next potential cluster
					f2fs_init_compress_ctx(&cc);
					cc.cluster_idx = NULL_CLUSTER;
				}
			} else { // Non-compressed file path
				int submitted_this_range = 0;
				// For non-compressed, f2fs_write_single_folio handles one dirty range.
				// It internally creates f2fs_io_info (fio) and calls f2fs_do_write_data_page.
				// It needs the folio (folio), and the dirty range.
				err = f2fs_write_single_folio(
					folio, &submitted_this_range, NULL,
					NULL, /* bio, last_block - managed by fio */
					wbc, io_type, 0 /*compr_blocks*/,
					false /*is_reclaim*/,
					pos,
					pos +
						r_len);
				// submitted_pages_this_folio += submitted_this_range; // f2fs_write_single_folio updates wbc->nr_to_write
				if (err) {
					if (err == -EAGAIN)
						wbc->nr_to_write = 0;
					goto handle_current_folio_error;
				}
			}
			pos +=
				r_len; // Advance for next iomap_find_dirty_range call
		} // End of while(iomap_find_dirty_range)

		// After processing all dirty ranges for this folio
		iomap_clear_range_dirty(folio, 0, folio_size(folio));

handle_current_folio_error: // Common error handling for the current folio
		folio_unlock(folio);
		if (fifs) {
			if (atomic_dec_and_test(&fifs->write_bytes_pending)) {
				folio_end_writeback(folio);
			}
		} else {
			// For single-page folio (no fifs), or if fifs alloc failed.
			// If no pages were submitted from this folio (e.g. error before submission,
			// or all dirty ranges were handled by compression context flush later),
			// then folio_end_writeback needs to be called.
			// This depends on whether f2fs_write_single_folio or f2fs_write_multi_pages
			// resulted in an actual bio submission for *this specific folio*.
			// If 'err' is set and no IO was submitted for this folio, or if no dirty ranges were found.
			// The original f2fs_write_cache_pages called release_pages.
			// Here, if err occurred, or if submitted_pages_this_folio (for non-compressed) is 0,
			// and for compressed, if this folio didn't complete a cluster that got written.
			// This part is tricky. The bias count with 'fifs' handles it well.
			// Without 'fifs', if no IO was submitted for this folio, folio_end_writeback is needed.
			// Let's assume if err is set, and it's not -EAGAIN, we might need it.
			// Or if no dirty ranges were processed for this folio.
			// For simplicity, if no 'fifs', and no error that implies IO (like -EAGAIN),
			// and if no actual submission happened for this folio (hard to track here without more state),
			// then folio_end_writeback.
			// The safest for non-fifs is if f2fs_write_single_folio only calls folio_end_writeback on actual IO completion.
			// If it fails before IO, then here.
			// If no dirty ranges, then here.
			if (!err &&
			    pos ==
				    current_folio_pos_in_file) { // No dirty ranges processed
				folio_end_writeback(folio);
			} else if (err && err != -EAGAIN) {
				// If an error stopped processing before any IO for this folio,
				// or if f2fs_write_single_folio failed before submitting.
				// This needs to be robust.
				// For now, rely on fifs or assume single-page folios' end_io handles it.
			}
		}
		if (err &&
		    err != -EAGAIN) { // If a real error occurred for this folio
			mapping_set_error(mapping, err);
			// writeback_iter will stop if wbc->sync_mode != WB_SYNC_NONE on error.
			// Or we can break here.
			break;
		}
		if (wbc->nr_to_write == 0 &&
		    wbc->sync_mode == WB_SYNC_NONE) { // Budget exhausted
			err = -EAGAIN; // Signal to VFS
			break;
		}
		err = 0; // Reset for next folio if current one was ok or -EAGAIN
	} // End of while(writeback_iter)

	if (is_compressed_file && !f2fs_cluster_is_empty(&cc)) {
		int submitted_compress = 0;
		int flush_err = f2fs_write_multi_pages(&cc, &submitted_compress,
						       wbc, io_type);
		wbc->nr_to_write -= submitted_compress;
		if (flush_err && !err) { // Prioritize earlier errors
			err = flush_err;
		}
	}

	if (is_compressed_file) {
		// Free cc.rpages and cc.cpages if they were allocated
		f2fs_destroy_compress_ctx(
			&cc,
			true); // true to free cpages, rpages also freed by page_array_free
	}

	// If err was -EAGAIN from writeback_iter (iter_err) or our loop, VFS expects 0.
	if (err == -EAGAIN || iter_err == -EAGAIN) {
		return 0;
	}
	return err ? err : iter_err;
}
```