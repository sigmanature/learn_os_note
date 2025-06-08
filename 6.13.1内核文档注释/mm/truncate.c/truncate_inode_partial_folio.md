```C
/**
 * truncate_inode_partial_folio - Handle partially truncated folios.
 * @folio: The folio potentially overlapping the truncation point.
 * @start: The byte offset within the file where truncation begins (new EOF).
 * @end: The byte offset within the file where truncation ends (inclusive, usually -1 for EOF).
 *
 * This function handles the case where the truncation point falls within a folio.
 * It zeroes the truncated part of the folio and attempts to split the folio
 * into smaller pieces so that the truncated parts can be managed independently
 * from the valid part.
 *
 * Returns: true if the folio was fully handled (either fully truncated or
 *          successfully split). Returns false if splitting failed on a dirty
 *          folio, indicating the caller should be careful not to discard the
 *          entire (unsplit) folio.
 */
bool truncate_inode_partial_folio(struct folio *folio, loff_t start, loff_t end)
{
	loff_t pos = folio_pos(folio); // Get the starting byte offset of the folio in the file.
	unsigned int offset, length;   // Offset within folio and length of the part to be truncated/zeroed.

	/* Calculate offset within the folio where truncation/zeroing begins. */
	if (pos < start)
		offset = start - pos; // Truncation starts partway into the folio.
	else
		offset = 0; // Truncation starts at or before the beginning of this folio.

	/* Calculate the length of the segment within this folio to be truncated/zeroed. */
	length = folio_size(folio); // Start assuming the whole folio from 'offset' onwards is truncated.
	if (pos + length <= (u64)end) // If the folio ends before or exactly at the truncation end point 'end'.
		length = length - offset; // The length to zero is from 'offset' to the folio's end.
	else
		// Folio straddles the truncation end point 'end'.
		// The length to zero is from 'offset' up to 'end'.
		length = end + 1 - pos - offset;

	// Wait for any ongoing writeback I/O on this folio to complete before modifying/splitting it.
	folio_wait_writeback(folio);

	// If the calculated length to truncate covers the entire folio (from offset 0).
	if (length == folio_size(folio) && offset == 0) {
		// Perform actions for fully truncating this folio (e.g., remove from LRU).
		truncate_inode_folio(folio->mapping, folio);
		return true; // Successfully handled as a full truncation.
	}

	/*
	 * Zero the part of the folio's memory that falls within the truncated range.
	 * We might zero parts we are about to discard via splitting, but doing it
	 * now simplifies logic and handles the case where splitting might fail.
	 */
	if (!mapping_inaccessible(folio->mapping)) // Check if mapping allows direct access (e.g., not DAX conflict).
		folio_zero_range(folio, offset, length); // Zero the memory range.

	// If the folio requires special handling upon invalidation (e.g., DAX).
	if (folio_needs_release(folio))
		folio_invalidate(folio, offset, length); // Call the mapping's invalidate operation.

	// If the folio is not large (order 0), no splitting is needed or possible.
	if (!folio_test_large(folio))
		return true; // Handled (by zeroing).

	/*
	 * Attempt to split the large folio into smaller (usually order-0) folios.
	 * This is crucial to isolate the truncated part from the valid part.
	 * We pass the folio itself, which implies splitting around the head page.
	 * The list parameter is NULL, so split folios go to LRU.
	 * split_folio typically performs a non-uniform split.
	 */
	if (split_folio(folio) == 0) // Returns 0 on successful split.
		return true; // Split successful, the folio is handled.

	/* --- Split Failed --- */
	// If splitting failed (e.g., due to GUP pins) and the folio is dirty...
	if (folio_test_dirty(folio))
		// ...we cannot easily discard it. Signal the caller that this folio
		// remains partially valid and unsplit.
		return false;

	// If split failed but the folio is not dirty (or became clean)...
	// ...attempt to treat it as a full truncation (remove from LRU etc.).
	// This might happen if the folio was fully zeroed above, or if the
	// valid part is also being discarded for other reasons.
	truncate_inode_folio(folio->mapping, folio);
	return true; // Handled (by attempting full truncation actions).
}

/**
 * split_folio - Split a large folio using non-uniform (buddy-like) splitting.
 * @folio: The large folio to split.
 *
 * This function attempts to split the given large folio into smaller pieces,
 * typically aiming for order-0 folios, using a non-uniform method similar
 * to the buddy allocator (splitting in half recursively).
 * It's a wrapper around __folio_split.
 *
 * Return: 0 on success, negative error code on failure.
 */
static inline int split_folio(struct folio *folio)
{
	// Call the internal split function.
	// new_order = 0: Target order for the piece containing 'split_at'.
	// split_at = &folio->page: Split centered around the head page.
	// lock_at = &folio->page: Leave the resulting folio containing the head page locked.
	// list = NULL: Put split-off folios onto LRU lists.
	// uniform_split = false: Perform non-uniform (buddy-style) split.
	return __folio_split(folio, 0, &folio->page, &folio->page, NULL, false);
}

/*
 * __folio_split: Internal function to split a folio.
 * @folio: Folio to split.
 * @new_order: Target order for the folio containing @split_at (for non-uniform),
 *             or the order for all resulting folios (for uniform).
 * @split_at: A page within the desired final folio piece (non-uniform).
 * @lock_at: A page within the original folio; the resulting folio containing
 *           this page will be left locked for the caller.
 * @list: List head to place split-off folios onto (if not NULL).
 * @uniform_split: True for uniform split, false for non-uniform (buddy-style).
 *
 * Prepares for splitting by handling locking, unmapping, and reference counting,
 * then calls __split_unmapped_folio to perform the actual split and page cache update.
 *
 * Return: 0 on success, negative error code on failure:
 *         -EINVAL: Invalid parameters (e.g., new_order >= current order).
 *         -EBUSY: Folio busy (writeback, huge zero page, mapping gone).
 *         -EAGAIN: Cannot split now (e.g., GUP pins, concurrent removal).
 */
static int __folio_split(struct folio *folio, unsigned int new_order,
		struct page *split_at, struct page *lock_at,
		struct list_head *list, bool uniform_split)
{
	// Get the deferred split queue associated with the folio's node/zone.
	struct deferred_split *ds_queue = get_deferred_split_queue(folio);
	// Prepare XArray state for potential page cache operations.
	XA_STATE(xas, &folio->mapping->i_pages, folio->index);
	// Check if the folio represents anonymous memory.
	bool is_anon = folio_test_anon(folio);
	struct address_space *mapping = NULL;
	struct anon_vma *anon_vma = NULL;
	int order = folio_order(folio); // Get the current order of the folio.
	int extra_pins, ret;
	pgoff_t end; // To store the end page index for file mappings.
	bool is_hzp; // Is it the huge zero page?

	// --- Prerequisite Checks ---
	VM_BUG_ON_FOLIO(!folio_test_locked(folio), folio); // Must be locked.
	VM_BUG_ON_FOLIO(!folio_test_large(folio), folio); // Must be large (order > 0).

	// Ensure split_at and lock_at pages belong to the folio being split.
	if (folio != page_folio(split_at) || folio != page_folio(lock_at))
		return -EINVAL;

	// Cannot split to an order greater than or equal to the current order.
	if (new_order >= folio_order(folio))
		return -EINVAL;

	// Check if the requested split type/order is supported for this folio type.
	if (uniform_split && !uniform_split_supported(folio, new_order, true))
		return -EINVAL;
	if (!uniform_split &&
	    !non_uniform_split_supported(folio, new_order, true))
		return -EINVAL;

	// Cannot split the global huge zero page.
	is_hzp = is_huge_zero_folio(folio);
	if (is_hzp) {
		pr_warn_ratelimited("Called split_huge_page for huge zero page\n");
		return -EBUSY;
	}

	// Cannot split if the folio is currently under writeback I/O.
	if (folio_test_writeback(folio))
		return -EBUSY;

	// --- Mapping/Locking Setup ---
	if (is_anon) {
		// Handle anonymous memory: Lock the associated anon_vma.
		anon_vma = folio_get_anon_vma(folio);
		if (!anon_vma) {
			ret = -EBUSY; // Failed to get anon_vma.
			goto out;
		}
		end = -1; // No concept of file EOF for anonymous.
		mapping = NULL;
		anon_vma_lock_write(anon_vma); // Lock for write to serialize against other splits/collapses.
	} else {
		// Handle file-backed memory.
		unsigned int min_order;
		gfp_t gfp;

		mapping = folio->mapping; // Get the address_space.

		// Check if mapping still exists (could be truncated).
		if (!mapping) {
			ret = -EBUSY;
			goto out;
		}

		// Ensure target order respects minimum order required by the mapping (if any).
		min_order = mapping_min_folio_order(folio->mapping);
		if (new_order < min_order) {
			VM_WARN_ONCE(1, "Cannot split mapped folio below min-order: %u",
				     min_order);
			ret = -EINVAL;
			goto out;
		}

		// Determine GFP context for potential memory allocations.
		gfp = current_gfp_context(mapping_gfp_mask(mapping) &
							GFP_RECLAIM_MASK);

		// Call filesystem hook to prepare for release/split (e.g., release fs-private data).
		if (!filemap_release_folio(folio, gfp)) {
			ret = -EBUSY; // Filesystem vetoed the split.
			goto out;
		}

		// For uniform split, pre-allocate XArray nodes if necessary.
		if (uniform_split) {
			xas_set_order(&xas, folio->index, new_order);
			xas_split_alloc(&xas, folio, folio_order(folio), gfp);
			if (xas_error(&xas)) {
				ret = xas_error(&xas);
				goto out;
			}
		}

		anon_vma = NULL;
		i_mmap_lock_read(mapping); // Lock mapping for reading (prevents VMA changes).

		// Get file size to handle pages beyond EOF during split.
		// Done here because i_size_read might not be safe under xas_lock.
		end = DIV_ROUND_UP(i_size_read(mapping->host), PAGE_SIZE);
		if (shmem_mapping(mapping)) // Special handling for shmem.
			end = shmem_fallocend(mapping->host, end);
	}

	// --- Prepare for Split ---
	// Racy check for extra pins (like GUP) before unmapping.
	if (!can_split_folio(folio, 1, &extra_pins)) {
		ret = -EAGAIN; // Cannot split due to pins.
		goto out_unlock;
	}

	// **Crucial Step**: Remove PTE mappings from all processes.
	unmap_folio(folio);

	// --- Perform Split (Critical Section) ---
	local_irq_disable(); // Disable local interrupts.
	if (mapping) {
		// Lock the page cache radix tree/XArray.
		xas_lock(&xas);
		xas_reset(&xas);
		// Verify the folio is still present at its index in the page cache.
		if (xas_load(&xas) != folio)
			goto fail; // Folio was removed concurrently.
	}

	// Lock the deferred split queue to prevent interference.
	spin_lock(&ds_queue->split_queue_lock);
	// **Crucial Step**: Attempt to freeze the folio's reference count.
	// This prevents refs from changing during the split metadata manipulation.
	// Checks against expected refs (1 base + extra_pins).
	if (folio_ref_freeze(folio, 1 + extra_pins)) {
		// Refcount frozen successfully.
		// Remove from deferred split queue if present.
		if (folio_order(folio) > 1 &&
		    !list_empty(&folio->_deferred_list)) {
			// ... (deferred split list management) ...
			list_del_init(&folio->_deferred_list);
		}
		spin_unlock(&ds_queue->split_queue_lock); // Unlock queue lock.

		// Update THP statistics if applicable.
		if (mapping) {
			// ... (THP stats update) ...
		}

		// **Call the function to perform the actual split logic.**
		ret = __split_unmapped_folio(folio, new_order,
				split_at, lock_at, list, end, &xas, mapping,
				uniform_split);
		// Note: __split_unmapped_folio unlocks xas if mapping is set.
		// IRQs remain disabled until after __split_unmapped_folio returns.

	} else {
		// Failed to freeze refcount (unexpected references).
		spin_unlock(&ds_queue->split_queue_lock);
fail: // Jump here if folio check failed too.
		if (mapping)
			xas_unlock(&xas); // Unlock XArray.
		local_irq_enable(); // Re-enable interrupts.
		// Attempt to remap the folio since split failed.
		remap_page(folio, folio_nr_pages(folio), 0);
		ret = -EAGAIN; // Signal failure due to unexpected refs/concurrent removal.
	}

	// --- Cleanup ---
out_unlock: // Jump here for earlier failures after acquiring mapping/anon_vma locks.
	if (anon_vma) {
		anon_vma_unlock_write(anon_vma);
		put_anon_vma(anon_vma); // Release anon_vma reference.
	}
	if (mapping)
		i_mmap_unlock_read(mapping); // Unlock mapping read lock.
out: // Final exit point.
	xas_destroy(&xas); // Clean up XArray state.
	// Update VM event counters for splitting.
	if (order == HPAGE_PMD_ORDER)
		count_vm_event(!ret ? THP_SPLIT_PAGE : THP_SPLIT_PAGE_FAILED);
	count_mthp_stat(order, !ret ? MTHP_STAT_SPLIT : MTHP_STAT_SPLIT_FAILED);
	return ret; // Return status code.
}


/**
 * __split_unmapped_folio - Perform the split on an unmapped folio.
 * @folio: The large folio to split (refcount frozen).
 * @new_order: Target order (see __folio_split).
 * @split_at: Page determining the target piece (non-uniform).
 * @lock_at: Page whose resulting folio should remain locked.
 * @list: List to add split-off folios to (or NULL for LRU).
 * @end: End page index of the file (-1 for anonymous).
 * @xas: Locked XArray state pointing to the folio's position in page cache.
 * @mapping: Address space of the folio (NULL for anonymous).
 * @uniform_split: True for uniform split, false for non-uniform.
 *
 * This function performs the core logic of splitting a folio whose refcount
 * is frozen and which is unmapped from process page tables. It modifies the
 * folio metadata, updates the page cache XArray, and handles LRU/list placement.
 * Assumes IRQs are disabled by the caller.
 *
 * Return: 0 on success, negative error code (e.g., -ENOMEM from xas_try_split).
 */
static int __split_unmapped_folio(struct folio *folio, int new_order,
		struct page *split_at, struct page *lock_at,
		struct list_head *list, pgoff_t end,
		struct xa_state *xas, struct address_space *mapping,
		bool uniform_split)
{
	struct lruvec *lruvec;
	struct address_space *swap_cache = NULL;
	// Keep track of the original folio (head of the compound page).
	struct folio *origin_folio = folio;
	// Pointer used to iterate through resulting folios.
	struct folio *next_folio = folio_next(folio); // Points just after the original folio initially.
	struct folio *new_folio; // Temporary folio pointer.
	struct folio *next;      // Temporary folio pointer for inner loop.
	int order = folio_order(folio); // Original order.
	int split_order; // Current order being split down to.
	// Starting order for the split loop.
	int start_order = uniform_split ? new_order : order - 1;
	int nr_dropped = 0; // Count of pages dropped because they are beyond EOF.
	int ret = 0; // Return value.
	bool stop_split = false; // Flag to stop splitting early on error.

	// Handle swap cache specific setup (requires locking swap cache's tree).
	if (folio_test_swapcache(folio)) {
		VM_BUG_ON(mapping); // Cannot be both file-mapped and swap cache.
		// Swap cache folios can only be uniformly split to order-0.
		if (!uniform_split || new_order != 0)
			return -EINVAL;
		swap_cache = swap_address_space(folio->swap);
		xa_lock(&swap_cache->i_pages); // Lock the swap cache's XArray.
	}

	// Update anonymous memory statistics.
	if (folio_test_anon(folio))
		mod_mthp_stat(order, MTHP_STAT_NR_ANON, -1);

	// Lock the LRU vector associated with the folio. Refcount is already frozen.
	lruvec = folio_lruvec_lock(folio);

	// Clear any lingering hardware poison status from the large folio.
	folio_clear_has_hwpoisoned(folio);

	/*
	 * Main splitting loop.
	 * For non-uniform: Iterates from order-1 down to new_order.
	 * For uniform: Runs only once with split_order == new_order.
	 */
	for (split_order = start_order;
	     split_order >= new_order && !stop_split;
	     split_order--) {
		int old_order = folio_order(folio); // Order *before* this split step.
		struct folio *release; // Folio being processed in the inner loop.
		// Marks the end of the folios produced by this split step.
		struct folio *end_folio = folio_next(folio);

		// Skip order-1 for anonymous memory (not supported).
		if (folio_test_anon(folio) && split_order == 1)
			continue;
		// For uniform split, only run the iteration where split_order matches new_order.
		if (uniform_split && split_order != new_order)
			continue;

		// Prepare XArray for the split (if file-mapped).
		if (mapping) {
			if (uniform_split) {
				// Uniform split pre-allocated nodes earlier, just update XArray state.
				xas_split(xas, folio, old_order);
			} else {
				// Non-uniform split: Try to allocate XArray nodes if needed.
				xas_set_order(xas, folio->index, split_order);
				xas_try_split(xas, folio, old_order);
				if (xas_error(xas)) {
					// Failed to allocate XArray nodes (e.g., -ENOMEM).
					ret = xas_error(xas);
					stop_split = true; // Stop further splitting.
					goto after_split; // Process results of *this* failed split attempt.
				}
			}
		}

		// Update memory cgroup references and page owner info for the split.
		folio_split_memcg_refs(folio, old_order, split_order);
		split_page_owner(&folio->page, old_order, split_order);
		pgalloc_tag_split(folio, old_order, split_order); // Update page allocator tags.

		// **Perform the low-level split of metadata within the compound page.**
		__split_folio_to_order(folio, old_order, split_order);

after_split: // Label jumped to on xas_try_split failure.
		/*
		 * Inner loop: Process the folios resulting from the *current* split step.
		 * 'folio' points to the first half, 'release' iterates through all pieces.
		 * 'end_folio' marks the end of the pieces from this step.
		 */
		for (release = folio; release != end_folio; release = next) {
			next = folio_next(release); // Get pointer to the *next* piece.

			/*
			 * For non-uniform split: If this piece ('release') contains the
			 * target page ('split_at') AND we haven't reached the final
			 * 'new_order' yet AND no error occurred ('stop_split' is false),
			 * then skip processing this piece now. It will become the 'folio'
			 * for the *next* iteration of the outer loop to be split further.
			 */
			if (release == page_folio(split_at)) {
				folio = release; // Update 'folio' for the next outer loop iteration.
				if (split_order != new_order && !stop_split)
					continue; // Skip processing, will be split again.
			}

			// Update statistics for anonymous folios.
			if (folio_test_anon(release)) {
				mod_mthp_stat(folio_order(release),
						MTHP_STAT_NR_ANON, 1);
			}

			/*
			 * Keep the very first folio ('origin_folio') frozen until the end.
			 * This prevents races where lookups might find the original folio
			 * before the page cache pointers are fully updated.
			 */
			if (release == origin_folio)
				continue; // Skip unfreezing the head folio for now.

			// Unfreeze the reference count for this split-off piece.
			// Add refs needed for page cache / swap cache entries.
			folio_ref_unfreeze(release, 1 +
					((mapping || swap_cache) ?
						folio_nr_pages(release) : 0));

			// Add the split-off folio to the LRU list or the provided list.
			lru_add_split_folio(origin_folio, release, lruvec,
					list);

			/* Handle pages beyond the end of the file (for file mappings). */
			if (mapping && release->index >= end) {
				// Page is past EOF, remove it from page cache.
				if (shmem_mapping(mapping)) // Special accounting for shmem.
					nr_dropped += folio_nr_pages(release);
				else if (folio_test_clear_dirty(release)) // Clear dirty if needed.
					folio_account_cleaned(release,
						inode_to_wb(mapping->host));
				// Remove from XArray (page cache).
				__filemap_remove_folio(release, NULL);
				// Release the references held by the page cache entry.
				folio_put_refs(release, folio_nr_pages(release));
			} else if (mapping) {
				// Page is valid, store its pointer in the page cache XArray.
				// *** This updates the page cache structure ***
				__xa_store(&mapping->i_pages,
						release->index, release, 0);
			} else if (swap_cache) {
				// Store pointer in the swap cache XArray.
				__xa_store(&swap_cache->i_pages,
						swap_cache_index(release->swap),
						release, 0);
			}
		} // End inner loop (processing pieces from one split step)
	} // End outer loop (splitting order by order)

	/*
	 * Now that all other page cache entries are updated, unfreeze the
	 * reference count of the original head folio.
	 */
	folio_ref_unfreeze(origin_folio, 1 +
		((mapping || swap_cache) ? folio_nr_pages(origin_folio) : 0));

	// Unlock the LRU list.
	unlock_page_lruvec(lruvec);

	// Unlock swap cache / page cache XArrays.
	if (swap_cache)
		xa_unlock(&swap_cache->i_pages);
	if (mapping)
		xa_unlock(&mapping->i_pages); // xas was locked in caller (__folio_split)

	// Re-enable local interrupts (were disabled in caller __folio_split).
	local_irq_enable();

	// Perform shmem uncharging if pages were dropped.
	if (nr_dropped)
		shmem_uncharge(mapping->host, nr_dropped);

	// Attempt to remap the pages after split (e.g., update PMD entries if needed).
	remap_page(origin_folio, 1 << order,
			folio_test_anon(origin_folio) ?
				RMP_USE_SHARED_ZEROPAGE : 0);

	/*
	 * Unlock all resulting folios EXCEPT the one containing 'lock_at'.
	 * The caller expects that specific folio piece to remain locked.
	 */
	for (new_folio = origin_folio; new_folio != next_folio; new_folio = next) {
		next = folio_next(new_folio);
		if (new_folio == page_folio(lock_at))
			continue; // Skip the one to be left locked.

		folio_unlock(new_folio); // Unlock the others.
		/*
		 * Release the reference held by this function for the split-off pieces
		 * (except the 'lock_at' one). This might free the page if no other
		 * references exist (e.g., if it wasn't added to page cache or LRU).
		 */
		free_page_and_swap_cache(&new_folio->page);
	}
	return ret; // Return status (0 on success, or error from xas_try_split).
}


/**
 * __split_folio_to_order - Low-level folio metadata split.
 * @folio: The folio piece to split (input/output, becomes first resulting piece).
 * @old_order: The order of the input @folio.
 * @new_order: The target order for the resulting pieces.
 *
 * Modifies the metadata within the compound page structure represented by @folio
 * to turn it into multiple smaller folios of order @new_order. It adjusts the
 * order of the input @folio and initializes the metadata (flags, mapping, index)
 * for the subsequent folio pieces derived from the tail pages.
 *
 * Assumes the folio is locked and refcount is frozen. Does NOT interact with
 * page cache XArray or LRU lists directly.
 */
static void __split_folio_to_order(struct folio *folio, int old_order,
		int new_order)
{
	long new_nr_pages = 1L << new_order; // Number of pages in each resulting folio.
	long nr_pages = 1L << old_order; // Total number of pages in the input folio.
	long i; // Loop counter for pages.

	/*
	 * Loop starts from the first page of the *second* resulting folio.
	 * The first resulting folio (pages 0 to new_nr_pages-1) is represented
	 * by the original 'folio' object, which already has the correct base
	 * metadata (flags, mapping, index). We only need to adjust its order later.
	 */
	for (i = new_nr_pages; i < nr_pages; i += new_nr_pages) {
		// Get pointer to the head page of the next resulting folio piece.
		struct page *new_head = &folio->page + i;

		/*
		 * Treat the 'struct page' at 'new_head' as a 'struct folio'.
		 * This works because folio is essentially page + metadata overlay.
		 * NOTE: It's not fully initialized as a folio head yet.
		 */
		struct folio *new_folio = (struct folio *)new_head;

		// Sanity check: mapcount should be -1 for tail pages becoming heads.
		VM_BUG_ON_PAGE(atomic_read(&new_folio->_mapcount) != -1, new_head);

		/* --- Initialize the new folio's metadata --- */

		// Clear flags that shouldn't be inherited or need re-evaluation.
		new_folio->flags &= ~PAGE_FLAGS_CHECK_AT_PREP;
		// Copy relevant flags from the original folio.
		new_folio->flags |= (folio->flags &
				((1L << PG_referenced) |
				 (1L << PG_swapbacked) | // Part of same swap entity
				 (1L << PG_swapcache) |  // Is in swap cache
				 (1L << PG_mlocked) |    // Inherit mlock status
				 (1L << PG_uptodate) |   // Inherit uptodate status
				 (1L << PG_active) |     // Inherit LRU active status
				 (1L << PG_workingset) | // Inherit workingset status
				 (1L << PG_locked) |     // Inherit lock status (will be unlocked later if not lock_at)
				 (1L << PG_unevictable)| // Inherit unevictable status
#ifdef CONFIG_ARCH_USES_PG_ARCH_2
				 (1L << PG_arch_2) |
#endif
#ifdef CONFIG_ARCH_USES_PG_ARCH_3
				 (1L << PG_arch_3) |
#endif
				 (1L << PG_dirty) |      // ** Inherit dirty status **
				 LRU_GEN_MASK | LRU_REFS_MASK)); // Inherit LRU generation/refs

		// ** Point the new folio piece to the same address_space mapping. **
		new_folio->mapping = folio->mapping;
		// Calculate the file offset (page index) for this new folio piece.
		new_folio->index = folio->index + i;

		// Sanity check and clear unexpected private data on tail pages.
		if (unlikely(new_folio->private)) {
			VM_WARN_ON_ONCE_PAGE(true, new_head);
			new_folio->private = NULL; // folio->private is NOT inherited.
		}

		// If in swap cache, calculate the correct swap entry offset.
		if (folio_test_swapcache(folio))
			new_folio->swap.val = folio->swap.val + i;

		/* Memory barrier: Ensure flag updates are visible before clearing PageTail. */
		smp_wmb();

		/*
		 * **Crucial Step**: Clear the PageTail flag on the 'new_head' page.
		 * This makes it the head page of a new (potentially compound) page/folio.
		 */
		clear_compound_head(new_head);
		// If the new order is > 0, mark it as a compound page head.
		if (new_order) {
			prep_compound_page(new_head, new_order);
			folio_set_large_rmappable(new_folio); // Mark as large folio
		}

		// Inherit accessed/idle status for LRU.
		if (folio_test_young(folio))
			folio_set_young(new_folio);
		if (folio_test_idle(folio))
			folio_set_idle(new_folio);
#ifdef CONFIG_MEMCG
		// Inherit memcg association.
		new_folio->memcg_data = folio->memcg_data;
#endif

		// Transfer NUMA node information (last CPU that touched it).
		folio_xchg_last_cpupid(new_folio, folio_last_cpupid(folio));
	} // End loop initializing subsequent folio pieces.

	/*
	 * Finally, adjust the order of the *first* folio piece (the original 'folio' object).
	 */
	if (new_order)
		// Set the new, smaller order if it's still a compound page.
		folio_set_order(folio, new_order);
	else
		// If the target order is 0, clear the compound flag entirely.
		ClearPageCompound(&folio->page);
}
```
