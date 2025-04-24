f2fs超级块的初始化逻辑,几乎承载了所有我们想要看的信息(比如xattr的大小,对数扇区大小等等)
```C
static int f2fs_fill_super(struct super_block *sb, void *data, int silent)
{
	struct f2fs_sb_info *sbi;
	struct f2fs_super_block *raw_super;
	struct inode *root;
	int err;
	bool skip_recovery = false, need_fsck = false;
	char *options = NULL;
	int recovery, i, valid_super_block;
	struct curseg_info *seg_i;
	int retry_cnt = 1;
#ifdef CONFIG_QUOTA
	bool quota_enabled = false;
#endif

try_onemore:
	err = -EINVAL;
	raw_super = NULL;
	valid_super_block = -1;
	recovery = 0;

	/* allocate memory for f2fs-specific super block info */
	sbi = kzalloc(sizeof(struct f2fs_sb_info), GFP_KERNEL);
	if (!sbi)
		return -ENOMEM;

	sbi->sb = sb;

	/* initialize locks within allocated memory */
	init_f2fs_rwsem(&sbi->gc_lock);
	mutex_init(&sbi->writepages);
	init_f2fs_rwsem(&sbi->cp_global_sem);
	init_f2fs_rwsem(&sbi->node_write);
	init_f2fs_rwsem(&sbi->node_change);
	spin_lock_init(&sbi->stat_lock);
	init_f2fs_rwsem(&sbi->cp_rwsem);
	init_f2fs_rwsem(&sbi->quota_sem);
	init_waitqueue_head(&sbi->cp_wait);
	spin_lock_init(&sbi->error_lock);

	for (i = 0; i < NR_INODE_TYPE; i++) {
		INIT_LIST_HEAD(&sbi->inode_list[i]);
		spin_lock_init(&sbi->inode_lock[i]);
	}
	mutex_init(&sbi->flush_lock);

	/* set a block size */
	if (unlikely(!sb_set_blocksize(sb, F2FS_BLKSIZE))) {
		f2fs_err(sbi, "unable to set blocksize");
		goto free_sbi;
	}

	err = read_raw_super_block(sbi, &raw_super, &valid_super_block,
								&recovery);
	if (err)
		goto free_sbi;

	sb->s_fs_info = sbi;
	sbi->raw_super = raw_super;

	INIT_WORK(&sbi->s_error_work, f2fs_record_error_work);
	memcpy(sbi->errors, raw_super->s_errors, MAX_F2FS_ERRORS);
	memcpy(sbi->stop_reason, raw_super->s_stop_reason, MAX_STOP_REASON);

	/* precompute checksum seed for metadata */
	if (f2fs_sb_has_inode_chksum(sbi))
		sbi->s_chksum_seed = f2fs_chksum(sbi, ~0, raw_super->uuid,
						sizeof(raw_super->uuid));

	default_options(sbi, false);
	/* parse mount options */
	options = kstrdup((const char *)data, GFP_KERNEL);
	if (data && !options) {
		err = -ENOMEM;
		goto free_sb_buf;
	}

	err = parse_options(sbi, options, false);
	if (err)
		goto free_options;

	err = f2fs_default_check(sbi);
	if (err)
		goto free_options;

	sb->s_maxbytes = max_file_blocks(NULL) <<
				le32_to_cpu(raw_super->log_blocksize);
	sb->s_max_links = F2FS_LINK_MAX;

	err = f2fs_setup_casefold(sbi);
	if (err)
		goto free_options;

#ifdef CONFIG_QUOTA
	sb->dq_op = &f2fs_quota_operations;
	sb->s_qcop = &f2fs_quotactl_ops;
	sb->s_quota_types = QTYPE_MASK_USR | QTYPE_MASK_GRP | QTYPE_MASK_PRJ;

	if (f2fs_sb_has_quota_ino(sbi)) {
		for (i = 0; i < MAXQUOTAS; i++) {
			if (f2fs_qf_ino(sbi->sb, i))
				sbi->nquota_files++;
		}
	}
#endif

	sb->s_op = &f2fs_sops;
#ifdef CONFIG_FS_ENCRYPTION
	sb->s_cop = &f2fs_cryptops;
#endif
#ifdef CONFIG_FS_VERITY
	sb->s_vop = &f2fs_verityops;
#endif
	sb->s_xattr = f2fs_xattr_handlers;
	sb->s_export_op = &f2fs_export_ops;
	sb->s_magic = F2FS_SUPER_MAGIC;
	sb->s_time_gran = 1;
	sb->s_flags = (sb->s_flags & ~SB_POSIXACL) |
		(test_opt(sbi, POSIX_ACL) ? SB_POSIXACL : 0);
	if (test_opt(sbi, INLINECRYPT))
		sb->s_flags |= SB_INLINECRYPT;

	if (test_opt(sbi, LAZYTIME))
		sb->s_flags |= SB_LAZYTIME;
	else
		sb->s_flags &= ~SB_LAZYTIME;

	super_set_uuid(sb, (void *) raw_super->uuid, sizeof(raw_super->uuid));
	super_set_sysfs_name_bdev(sb);
	sb->s_iflags |= SB_I_CGROUPWB;

	/* init f2fs-specific super block info */
	sbi->valid_super_block = valid_super_block;

	/* disallow all the data/node/meta page writes */
	set_sbi_flag(sbi, SBI_POR_DOING);

	err = f2fs_init_write_merge_io(sbi);
	if (err)
		goto free_bio_info;

	init_sb_info(sbi);

	err = f2fs_init_iostat(sbi);
	if (err)
		goto free_bio_info;

	err = init_percpu_info(sbi);
	if (err)
		goto free_iostat;

	/* init per sbi slab cache */
	err = f2fs_init_xattr_caches(sbi);
	if (err)
		goto free_percpu;
	err = f2fs_init_page_array_cache(sbi);
	if (err)
		goto free_xattr_cache;

	/* get an inode for meta space */
	sbi->meta_inode = f2fs_iget(sb, F2FS_META_INO(sbi));/*可以看见元数据inode的构造也用的是f2fs_iget函数*/
	if (IS_ERR(sbi->meta_inode)) {
		f2fs_err(sbi, "Failed to read F2FS meta data inode");
		err = PTR_ERR(sbi->meta_inode);
		goto free_page_array_cache;
	}

	err = f2fs_get_valid_checkpoint(sbi);
	if (err) {
		f2fs_err(sbi, "Failed to get valid F2FS checkpoint");
		goto free_meta_inode;
	}

	if (__is_set_ckpt_flags(F2FS_CKPT(sbi), CP_QUOTA_NEED_FSCK_FLAG))
		set_sbi_flag(sbi, SBI_QUOTA_NEED_REPAIR);
	if (__is_set_ckpt_flags(F2FS_CKPT(sbi), CP_DISABLED_QUICK_FLAG)) {
		set_sbi_flag(sbi, SBI_CP_DISABLED_QUICK);
		sbi->interval_time[DISABLE_TIME] = DEF_DISABLE_QUICK_INTERVAL;
	}

	if (__is_set_ckpt_flags(F2FS_CKPT(sbi), CP_FSCK_FLAG))
		set_sbi_flag(sbi, SBI_NEED_FSCK);

	/* Initialize device list */
	err = f2fs_scan_devices(sbi);
	if (err) {
		f2fs_err(sbi, "Failed to find devices");
		goto free_devices;
	}

	err = f2fs_init_post_read_wq(sbi);
	if (err) {
		f2fs_err(sbi, "Failed to initialize post read workqueue");
		goto free_devices;
	}

	sbi->total_valid_node_count =
				le32_to_cpu(sbi->ckpt->valid_node_count);
	percpu_counter_set(&sbi->total_valid_inode_count,
				le32_to_cpu(sbi->ckpt->valid_inode_count));
	sbi->user_block_count = le64_to_cpu(sbi->ckpt->user_block_count);
	sbi->total_valid_block_count =
				le64_to_cpu(sbi->ckpt->valid_block_count);
	sbi->last_valid_block_count = sbi->total_valid_block_count;
	sbi->reserved_blocks = 0;
	sbi->current_reserved_blocks = 0;
	limit_reserve_root(sbi);
	adjust_unusable_cap_perc(sbi);

	f2fs_init_extent_cache_info(sbi);

	f2fs_init_ino_entry_info(sbi);

	f2fs_init_fsync_node_info(sbi);

	/* setup checkpoint request control and start checkpoint issue thread */
	f2fs_init_ckpt_req_control(sbi);
	if (!f2fs_readonly(sb) && !test_opt(sbi, DISABLE_CHECKPOINT) &&
			test_opt(sbi, MERGE_CHECKPOINT)) {
		err = f2fs_start_ckpt_thread(sbi);
		if (err) {
			f2fs_err(sbi,
			    "Failed to start F2FS issue_checkpoint_thread (%d)",
			    err);
			goto stop_ckpt_thread;
		}
	}

	/* setup f2fs internal modules */
	err = f2fs_build_segment_manager(sbi);
	if (err) {
		f2fs_err(sbi, "Failed to initialize F2FS segment manager (%d)",
			 err);
		goto free_sm;
	}
	err = f2fs_build_node_manager(sbi);
	if (err) {
		f2fs_err(sbi, "Failed to initialize F2FS node manager (%d)",
			 err);
		goto free_nm;
	}

	/* For write statistics */
	sbi->sectors_written_start = f2fs_get_sectors_written(sbi);

	/* get segno of first zoned block device */
	sbi->first_zoned_segno = get_first_zoned_segno(sbi);

	/* Read accumulated write IO statistics if exists */
	seg_i = CURSEG_I(sbi, CURSEG_HOT_NODE);
	if (__exist_node_summaries(sbi))
		sbi->kbytes_written =
			le64_to_cpu(seg_i->journal->info.kbytes_written);

	f2fs_build_gc_manager(sbi);

	err = f2fs_build_stats(sbi);
	if (err)
		goto free_nm;

	/* get an inode for node space */
	sbi->node_inode = f2fs_iget(sb, F2FS_NODE_INO(sbi));
	if (IS_ERR(sbi->node_inode)) {
		f2fs_err(sbi, "Failed to read node inode");
		err = PTR_ERR(sbi->node_inode);
		goto free_stats;
	}

	/* read root inode and dentry */
	root = f2fs_iget(sb, F2FS_ROOT_INO(sbi));
	if (IS_ERR(root)) {
		f2fs_err(sbi, "Failed to read root inode");
		err = PTR_ERR(root);
		goto free_node_inode;
	}
	if (!S_ISDIR(root->i_mode) || !root->i_blocks ||
			!root->i_size || !root->i_nlink) {
		iput(root);
		err = -EINVAL;
		goto free_node_inode;
	}

	generic_set_sb_d_ops(sb);
	sb->s_root = d_make_root(root); /* allocate root dentry */
	if (!sb->s_root) {
		err = -ENOMEM;
		goto free_node_inode;
	}

	err = f2fs_init_compress_inode(sbi);
	if (err)
		goto free_root_inode;

	err = f2fs_register_sysfs(sbi);
	if (err)
		goto free_compress_inode;

	sbi->umount_lock_holder = current;
#ifdef CONFIG_QUOTA
	/* Enable quota usage during mount */
	if (f2fs_sb_has_quota_ino(sbi) && !f2fs_readonly(sb)) {
		err = f2fs_enable_quotas(sb);
		if (err)
			f2fs_err(sbi, "Cannot turn on quotas: error %d", err);
	}

	quota_enabled = f2fs_recover_quota_begin(sbi);
#endif
	/* if there are any orphan inodes, free them */
	err = f2fs_recover_orphan_inodes(sbi);
	if (err)
		goto free_meta;

	if (unlikely(is_set_ckpt_flags(sbi, CP_DISABLED_FLAG))) {
		skip_recovery = true;
		goto reset_checkpoint;
	}

	/* recover fsynced data */
	if (!test_opt(sbi, DISABLE_ROLL_FORWARD) &&
			!test_opt(sbi, NORECOVERY)) {
		/*
		 * mount should be failed, when device has readonly mode, and
		 * previous checkpoint was not done by clean system shutdown.
		 */
		if (f2fs_hw_is_readonly(sbi)) {
			if (!is_set_ckpt_flags(sbi, CP_UMOUNT_FLAG)) {
				err = f2fs_recover_fsync_data(sbi, true);
				if (err > 0) {
					err = -EROFS;
					f2fs_err(sbi, "Need to recover fsync data, but "
						"write access unavailable, please try "
						"mount w/ disable_roll_forward or norecovery");
				}
				if (err < 0)
					goto free_meta;
			}
			f2fs_info(sbi, "write access unavailable, skipping recovery");
			goto reset_checkpoint;
		}

		if (need_fsck)
			set_sbi_flag(sbi, SBI_NEED_FSCK);

		if (skip_recovery)
			goto reset_checkpoint;

		err = f2fs_recover_fsync_data(sbi, false);
		if (err < 0) {
			if (err != -ENOMEM)
				skip_recovery = true;
			need_fsck = true;
			f2fs_err(sbi, "Cannot recover all fsync data errno=%d",
				 err);
			goto free_meta;
		}
	} else {
		err = f2fs_recover_fsync_data(sbi, true);

		if (!f2fs_readonly(sb) && err > 0) {
			err = -EINVAL;
			f2fs_err(sbi, "Need to recover fsync data");
			goto free_meta;
		}
	}

reset_checkpoint:
#ifdef CONFIG_QUOTA
	f2fs_recover_quota_end(sbi, quota_enabled);
#endif
	/*
	 * If the f2fs is not readonly and fsync data recovery succeeds,
	 * write pointer consistency of cursegs and other zones are already
	 * checked and fixed during recovery. However, if recovery fails,
	 * write pointers are left untouched, and retry-mount should check
	 * them here.
	 */
	if (skip_recovery)
		err = f2fs_check_and_fix_write_pointer(sbi);
	if (err)
		goto free_meta;

	/* f2fs_recover_fsync_data() cleared this already */
	clear_sbi_flag(sbi, SBI_POR_DOING);

	err = f2fs_init_inmem_curseg(sbi);
	if (err)
		goto sync_free_meta;

	if (test_opt(sbi, DISABLE_CHECKPOINT)) {
		err = f2fs_disable_checkpoint(sbi);
		if (err)
			goto sync_free_meta;
	} else if (is_set_ckpt_flags(sbi, CP_DISABLED_FLAG)) {
		f2fs_enable_checkpoint(sbi);
	}

	/*
	 * If filesystem is not mounted as read-only then
	 * do start the gc_thread.
	 */
	if ((F2FS_OPTION(sbi).bggc_mode != BGGC_MODE_OFF ||
		test_opt(sbi, GC_MERGE)) && !f2fs_readonly(sb)) {
		/* After POR, we can run background GC thread.*/
		err = f2fs_start_gc_thread(sbi);
		if (err)
			goto sync_free_meta;
	}
	kvfree(options);

	/* recover broken superblock */
	if (recovery) {
		err = f2fs_commit_super(sbi, true);
		f2fs_info(sbi, "Try to recover %dth superblock, ret: %d",
			  sbi->valid_super_block ? 1 : 2, err);
	}

	f2fs_join_shrinker(sbi);

	f2fs_tuning_parameters(sbi);

	f2fs_notice(sbi, "Mounted with checkpoint version = %llx",
		    cur_cp_version(F2FS_CKPT(sbi)));
	f2fs_update_time(sbi, CP_TIME);
	f2fs_update_time(sbi, REQ_TIME);
	clear_sbi_flag(sbi, SBI_CP_DISABLED_QUICK);

	sbi->umount_lock_holder = NULL;
	return 0;

sync_free_meta:
	/* safe to flush all the data */
	sync_filesystem(sbi->sb);
	retry_cnt = 0;

free_meta:
#ifdef CONFIG_QUOTA
	f2fs_truncate_quota_inode_pages(sb);
	if (f2fs_sb_has_quota_ino(sbi) && !f2fs_readonly(sb))
		f2fs_quota_off_umount(sbi->sb);
#endif
	/*
	 * Some dirty meta pages can be produced by f2fs_recover_orphan_inodes()
	 * failed by EIO. Then, iput(node_inode) can trigger balance_fs_bg()
	 * followed by f2fs_write_checkpoint() through f2fs_write_node_pages(), which
	 * falls into an infinite loop in f2fs_sync_meta_pages().
	 */
	truncate_inode_pages_final(META_MAPPING(sbi));
	/* evict some inodes being cached by GC */
	evict_inodes(sb);
	f2fs_unregister_sysfs(sbi);
free_compress_inode:
	f2fs_destroy_compress_inode(sbi);
free_root_inode:
	dput(sb->s_root);
	sb->s_root = NULL;
free_node_inode:
	f2fs_release_ino_entry(sbi, true);
	truncate_inode_pages_final(NODE_MAPPING(sbi));
	iput(sbi->node_inode);
	sbi->node_inode = NULL;
free_stats:
	f2fs_destroy_stats(sbi);
free_nm:
	/* stop discard thread before destroying node manager */
	f2fs_stop_discard_thread(sbi);
	f2fs_destroy_node_manager(sbi);
free_sm:
	f2fs_destroy_segment_manager(sbi);
stop_ckpt_thread:
	f2fs_stop_ckpt_thread(sbi);
	/* flush s_error_work before sbi destroy */
	flush_work(&sbi->s_error_work);
	f2fs_destroy_post_read_wq(sbi);
free_devices:
	destroy_device_list(sbi);
	kvfree(sbi->ckpt);
free_meta_inode:
	make_bad_inode(sbi->meta_inode);
	iput(sbi->meta_inode);
	sbi->meta_inode = NULL;
free_page_array_cache:
	f2fs_destroy_page_array_cache(sbi);
free_xattr_cache:
	f2fs_destroy_xattr_caches(sbi);
free_percpu:
	destroy_percpu_info(sbi);
free_iostat:
	f2fs_destroy_iostat(sbi);
free_bio_info:
	for (i = 0; i < NR_PAGE_TYPE; i++)
		kvfree(sbi->write_io[i]);

#if IS_ENABLED(CONFIG_UNICODE)
	utf8_unload(sb->s_encoding);
	sb->s_encoding = NULL;
#endif
free_options:
#ifdef CONFIG_QUOTA
	for (i = 0; i < MAXQUOTAS; i++)
		kfree(F2FS_OPTION(sbi).s_qf_names[i]);
#endif
	fscrypt_free_dummy_policy(&F2FS_OPTION(sbi).dummy_enc_policy);
	kvfree(options);
free_sb_buf:
	kfree(raw_super);
free_sbi:
	kfree(sbi);
	sb->s_fs_info = NULL;

	/* give only one another chance */
	if (retry_cnt > 0 && skip_recovery) {
		retry_cnt--;
		shrink_dcache_sb(sb);
		goto try_onemore;
	}
	return err;
}
```