```C
/**
 * __try_merge_extent_node - Attempt to merge a new extent with adjacent ones.
 * @sbi: Superblock info.
 * @et: The extent tree being modified.
 * @ei: The new extent info to be potentially merged.
 * @prev_ex: The potential preceding extent node in the tree.
 * @next_ex: The potential succeeding extent node in the tree.
 *
 * Checks if 'ei' can be merged with 'prev_ex' (back merge) and/or 'next_ex'
 * (front merge). If merges occur, updates the existing node(s) and releases
 * any node that becomes fully merged into another. Updates LRU list and
 * largest extent info if a merge happens.
 *
 * Returns: The resulting merged extent_node, or NULL if no merge occurred.
 */
static struct extent_node *__try_merge_extent_node(struct f2fs_sb_info *sbi,
				struct extent_tree *et, struct extent_info *ei,
				struct extent_node *prev_ex,
				struct extent_node *next_ex)
{
	// Get global extent tree info (for LRU list) based on type
	struct extent_tree_info *eti = &sbi->extent_tree[et->type];
	struct extent_node *en = NULL; // Pointer to the resulting merged node

	// Check for back merge: Can 'ei' be appended to 'prev_ex'?
	if (prev_ex && __is_back_mergeable(ei, &prev_ex->ei, et->type)) {
		// Yes: Extend the length of prev_ex
		prev_ex->ei.len += ei->len;
		// Point 'ei' to the updated info in prev_ex for potential front merge check
		ei = &prev_ex->ei;
		// Mark prev_ex as the current resulting node
		en = prev_ex;
		// Note: If compression is used, __is_back_mergeable ensures c_len constraints are met.
		// If ei->c_len was > 0, it might be reset here unless the merge logic preserves it.
		// However, __is_extent_mergeable prevents merging partial compressed clusters.
	}

	// Check for front merge: Can 'next_ex' be appended to 'ei'?
	// 'ei' might now be pointing to prev_ex->ei if a back merge happened.
	if (next_ex && __is_front_mergeable(ei, &next_ex->ei, et->type)) {
		// Yes: Update next_ex's starting offset and length
		next_ex->ei.fofs = ei->fofs;
		next_ex->ei.len += ei->len;
		// If it's a read extent, update the starting block address as well
		if (et->type == EX_READ)
			next_ex->ei.blk = ei->blk;
		// If a back merge also happened ('en' is set, points to prev_ex),
		// it means prev_ex is now merged into next_ex, so release prev_ex.
		if (en)
			__release_extent_node(sbi, et, prev_ex);

		// Mark next_ex as the final resulting node
		en = next_ex;
		// Similar compression considerations as for back merge.
	}

	// If no merge occurred, 'en' is still NULL
	if (!en)
		return NULL;

	// If a merge happened, update the largest extent info if this merged node is larger
	__try_update_largest_extent(et, en);

	// Update LRU status: Move the merged node to the tail of the global list
	spin_lock(&eti->extent_lock); // Protect the global list
	if (!list_empty(&en->list)) { // Ensure node is actually on a list
		list_move_tail(&en->list, &eti->extent_list); // Move to end (most recently used)
		et->cached_en = en; // Update the tree's cached pointer
	}
	spin_unlock(&eti->extent_lock); // Release list lock
	return en; // Return the merged node
}

```