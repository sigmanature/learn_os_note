```C
static void __update_extent_tree_range(struct inode *inode,
			struct extent_info *tei, enum extent_type type)
{
	struct f2fs_sb_info *sbi = F2FS_I_SB(inode);
	struct extent_tree *et = F2FS_I(inode)->extent_tree[type];
	struct extent_node *en = NULL, *en1 = NULL;
	struct extent_node *prev_en = NULL, *next_en = NULL;
	struct extent_info ei, dei, prev;
	struct rb_node **insert_p = NULL, *insert_parent = NULL;
	unsigned int fofs = tei->fofs, len = tei->len;
	unsigned int end = fofs + len;
	bool updated = false;
	bool leftmost = false;

	if (!et)
		return;

	if (type == EX_READ)
		trace_f2fs_update_read_extent_tree_range(inode, fofs, len,
						tei->blk, 0);
	else if (type == EX_BLOCK_AGE)
		trace_f2fs_update_age_extent_tree_range(inode, fofs, len,
						tei->age, tei->last_blocks);

	write_lock(&et->lock);

	if (type == EX_READ) {
		if (is_inode_flag_set(inode, FI_NO_EXTENT)) {
			write_unlock(&et->lock);
			return;
		}

		prev = et->largest;
		dei.len = 0;

		/*
		 * drop largest extent before lookup, in case it's already
		 * been shrunk from extent tree
		 */
		__drop_largest_extent(et, fofs, len);
	}

	/* 1. lookup first extent node in range [fofs, fofs + len - 1] */
	en = __lookup_extent_node_ret(&et->root,
					et->cached_en, fofs,
					&prev_en, &next_en,
					&insert_p, &insert_parent,
					&leftmost);
	if (!en)
		en = next_en;

	/* 2. invalidate all extent nodes in range [fofs, fofs + len - 1] */
	while (en && en->ei.fofs < end) {
		unsigned int org_end;
		int parts = 0;	/* # of parts current extent split into */

		next_en = en1 = NULL;

		dei = en->ei;
		org_end = dei.fofs + dei.len;
		f2fs_bug_on(sbi, fofs >= org_end);

		if (fofs > dei.fofs && (type != EX_READ ||
				fofs - dei.fofs >= F2FS_MIN_EXTENT_LEN)) {
			en->ei.len = fofs - en->ei.fofs;
			prev_en = en;
			parts = 1;
		}

		if (end < org_end && (type != EX_READ ||
			(org_end - end >= F2FS_MIN_EXTENT_LEN &&
			atomic_read(&et->node_cnt) <
					sbi->max_read_extent_count))) {
			if (parts) {
				__set_extent_info(&ei,
					end, org_end - end,
					end - dei.fofs + dei.blk, false,
					dei.age, dei.last_blocks,
					type);
				en1 = __insert_extent_tree(sbi, et, &ei,
							NULL, NULL, true);
				next_en = en1;
			} else {
				__set_extent_info(&en->ei,
					end, en->ei.len - (end - dei.fofs),
					en->ei.blk + (end - dei.fofs), true,
					dei.age, dei.last_blocks,
					type);
				next_en = en;
			}
			parts++;
		}

		if (!next_en) {
			struct rb_node *node = rb_next(&en->rb_node);

			next_en = rb_entry_safe(node, struct extent_node,
						rb_node);
		}

		if (parts)
			__try_update_largest_extent(et, en);
		else
			__release_extent_node(sbi, et, en);

		/*
		 * if original extent is split into zero or two parts, extent
		 * tree has been altered by deletion or insertion, therefore
		 * invalidate pointers regard to tree.
		 */
		if (parts != 1) {
			insert_p = NULL;
			insert_parent = NULL;
		}
		en = next_en;
	}

	if (type == EX_BLOCK_AGE)
		goto update_age_extent_cache;

	/* 3. update extent in read extent cache */
	BUG_ON(type != EX_READ);

	if (tei->blk) {
		__set_extent_info(&ei, fofs, len, tei->blk, false,
				  0, 0, EX_READ);
		if (!__try_merge_extent_node(sbi, et, &ei, prev_en, next_en))
			__insert_extent_tree(sbi, et, &ei,
					insert_p, insert_parent, leftmost);

		/* give up extent_cache, if split and small updates happen */
		if (dei.len >= 1 &&
				prev.len < F2FS_MIN_EXTENT_LEN &&
				et->largest.len < F2FS_MIN_EXTENT_LEN) {
			et->largest.len = 0;
			et->largest_updated = true;
			set_inode_flag(inode, FI_NO_EXTENT);
		}
	}

	if (et->largest_updated) {
		et->largest_updated = false;
		updated = true;
	}
	goto out_read_extent_cache;
update_age_extent_cache:
	if (!tei->last_blocks)
		goto out_read_extent_cache;

	__set_extent_info(&ei, fofs, len, 0, false,
			tei->age, tei->last_blocks, EX_BLOCK_AGE);
	if (!__try_merge_extent_node(sbi, et, &ei, prev_en, next_en))
		__insert_extent_tree(sbi, et, &ei,
					insert_p, insert_parent, leftmost);
out_read_extent_cache:
	write_unlock(&et->lock);

	if (is_inode_flag_set(inode, FI_NO_EXTENT))
		__destroy_extent_node(inode, EX_READ);

	if (updated)
		f2fs_mark_inode_dirty_sync(inode, true);
}
```