**相关结构体**
[swap_info_struct]()
`free_swap_and_cache_nr`函数的背景诞生于Ryan Robert的支持large folios被换出时不分裂的补丁代码。
它的实现是先减`swap_entry`的引用计数再释放。但是在新版本的内核似乎已经被重构得比较成熟了?
```C
static bool swap_entries_put_map(struct swap_info_struct *si,
				 swp_entry_t entry, int nr)
{
	unsigned long offset = swp_offset(entry);
	struct swap_cluster_info *ci;
	bool has_cache = false;
	unsigned char count;
	int i;

	if (nr <= 1)
		goto fallback;
	count = swap_count(data_race(si->swap_map[offset]));
	if (count != 1 && count != SWAP_MAP_SHMEM)
		goto fallback;

	ci = lock_cluster(si, offset);
	if (!swap_is_last_map(si, offset, nr, &has_cache)) {
		goto locked_fallback;
	}
	if (!has_cache)
		swap_entries_free(si, ci, entry, nr);
	else
		for (i = 0; i < nr; i++)
			WRITE_ONCE(si->swap_map[offset + i], SWAP_HAS_CACHE);
	unlock_cluster(ci);

	return has_cache;

fallback:
	ci = lock_cluster(si, offset);
locked_fallback:
	for (i = 0; i < nr; i++, entry.val++) {
		count = swap_entry_put_locked(si, ci, entry, 1);
		if (count == SWAP_HAS_CACHE)
			has_cache = true;
	}
	unlock_cluster(ci);
	return has_cache;

}
/*
 * Only functions with "_nr" suffix are able to free entries spanning
 * cross multi clusters, so ensure the range is within a single cluster
 * when freeing entries with functions without "_nr" suffix.
 */
static bool swap_entries_put_map_nr(struct swap_info_struct *si,
				    swp_entry_t entry, int nr)
{
	int cluster_nr, cluster_rest;
	unsigned long offset = swp_offset(entry);
	bool has_cache = false;

	cluster_rest = SWAPFILE_CLUSTER - offset % SWAPFILE_CLUSTER;
	while (nr) {
		cluster_nr = min(nr, cluster_rest);
		has_cache |= swap_entries_put_map(si, entry, cluster_nr);
		cluster_rest = SWAPFILE_CLUSTER;
		nr -= cluster_nr;
		entry.val += cluster_nr;
	}

	return has_cache;
}
/**
 * free_swap_and_cache_nr() - Release reference on range of swap entries and
 *                            reclaim their cache if no more references remain.
 * @entry: First entry of range.
 * @nr: Number of entries in range.
 *
 * For each swap entry in the contiguous range, release a reference. If any swap
 * entries become free, try to reclaim their underlying folios, if present. The
 * offset range is defined by [entry.offset, entry.offset + nr).
 */
/**
 * free_swap_and_cache_nr() - Release reference on range of swap entries and
 *                            reclaim their cache if no more references remain.
 * @entry: First entry of range.
 * @nr: Number of entries in range.
 *
 * For each swap entry in the contiguous range, release a reference. If any swap
 * entries become free, try to reclaim their underlying folios, if present. The
 * offset range is defined by [entry.offset, entry.offset + nr).
 */
void free_swap_and_cache_nr(swp_entry_t entry, int nr)
{
	const unsigned long start_offset = swp_offset(entry);
	const unsigned long end_offset = start_offset + nr;
	struct swap_info_struct *si;
	bool any_only_cache = false;
	unsigned long offset;

	si = get_swap_device(entry);
	if (!si)
		return;

	if (WARN_ON(end_offset > si->max))
		goto out;

	/*
	 * First free all entries in the range.
	 */
	any_only_cache = swap_entries_put_map_nr(si, entry, nr);

	/*
	 * Short-circuit the below loop if none of the entries had their
	 * reference drop to zero.
	 */
	if (!any_only_cache)
		goto out;

	/*
	 * Now go back over the range trying to reclaim the swap cache.
	 */
	for (offset = start_offset; offset < end_offset; offset += nr) {
		nr = 1;
		if (READ_ONCE(si->swap_map[offset]) == SWAP_HAS_CACHE) {
			/*
			 * Folios are always naturally aligned in swap so
			 * advance forward to the next boundary. Zero means no
			 * folio was found for the swap entry, so advance by 1
			 * in this case. Negative value means folio was found
			 * but could not be reclaimed. Here we can still advance
			 * to the next boundary.
			 */
			nr = __try_to_reclaim_swap(si, offset,
						   TTRS_UNMAPPED | TTRS_FULL);
			if (nr == 0)
				nr = 1;
			else if (nr < 0)
				nr = -nr;
			nr = ALIGN(offset + 1, nr) - offset;
		}
	}

out:
	put_swap_device(si);
}
```