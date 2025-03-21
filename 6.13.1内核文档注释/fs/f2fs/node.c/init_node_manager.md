```C
static int init_node_manager(struct f2fs_sb_info *sbi)
{
	struct f2fs_super_block *sb_raw = F2FS_RAW_SUPER(sbi);
	struct f2fs_nm_info *nm_i = NM_I(sbi);
	unsigned char *version_bitmap;
	unsigned int nat_segs;
	int err;

	nm_i->nat_blkaddr = le32_to_cpu(sb_raw->nat_blkaddr);

	/* segment_count_nat includes pair segment so divide to 2. */
	nat_segs = le32_to_cpu(sb_raw->segment_count_nat) >> 1;
	nm_i->nat_blocks = nat_segs << le32_to_cpu(sb_raw->log_blocks_per_seg);
	nm_i->max_nid = NAT_ENTRY_PER_BLOCK * nm_i->nat_blocks;/*重要 计算最大nid个数的逻辑* 就是每个块中NAT表项的数量*有的nat块数 也就是说这是一个动态的数字/

	/* not used nids: 0, node, meta, (and root counted as valid node) */
	nm_i->available_nids = nm_i->max_nid - sbi->total_valid_node_count -
						F2FS_RESERVED_NODE_NUM;
	nm_i->nid_cnt[FREE_NID] = 0;
	nm_i->nid_cnt[PREALLOC_NID] = 0;
	nm_i->ram_thresh = DEF_RAM_THRESHOLD;
	nm_i->ra_nid_pages = DEF_RA_NID_PAGES;
	nm_i->dirty_nats_ratio = DEF_DIRTY_NAT_RATIO_THRESHOLD;
	nm_i->max_rf_node_blocks = DEF_RF_NODE_BLOCKS;

	INIT_RADIX_TREE(&nm_i->free_nid_root, GFP_ATOMIC);
	INIT_LIST_HEAD(&nm_i->free_nid_list);
	INIT_RADIX_TREE(&nm_i->nat_root, GFP_NOIO);
	INIT_RADIX_TREE(&nm_i->nat_set_root, GFP_NOIO);
	INIT_LIST_HEAD(&nm_i->nat_entries);
	spin_lock_init(&nm_i->nat_list_lock);

	mutex_init(&nm_i->build_lock);
	spin_lock_init(&nm_i->nid_list_lock);
	init_f2fs_rwsem(&nm_i->nat_tree_lock);

	nm_i->next_scan_nid = le32_to_cpu(sbi->ckpt->next_free_nid);
	nm_i->bitmap_size = __bitmap_size(sbi, NAT_BITMAP);
	version_bitmap = __bitmap_ptr(sbi, NAT_BITMAP);
	nm_i->nat_bitmap = kmemdup(version_bitmap, nm_i->bitmap_size,
					GFP_KERNEL);
	if (!nm_i->nat_bitmap)
		return -ENOMEM;

	err = __get_nat_bitmaps(sbi);
	if (err)
		return err;

#ifdef CONFIG_F2FS_CHECK_FS
	nm_i->nat_bitmap_mir = kmemdup(version_bitmap, nm_i->bitmap_size,
					GFP_KERNEL);
	if (!nm_i->nat_bitmap_mir)
		return -ENOMEM;
#endif

	return 0;
}
```