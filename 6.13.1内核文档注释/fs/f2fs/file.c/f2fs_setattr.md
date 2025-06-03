linux truncate系统调用会走到具体文件系统inode->ops中的setattr函数调用。从名字上看不出来,但确实这个函数执行了实际的truncate操作。
**相关函数**
* [page_cache_ra_order](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/readahead.c/page_cache_ra_order.md)
* [truncate_setsize](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/readahead.c/truncate_setsize.md)
```C
// Main entry point for setting inode attributes in F2FS
int f2fs_setattr(struct mnt_idmap *idmap, struct dentry *dentry,
		 struct iattr *attr)
{
	struct inode *inode = d_inode(dentry); // Get inode from dentry
	struct f2fs_inode_info *fi = F2FS_I(inode); // Get F2FS specific inode info
	int err;

	// Check for filesystem-wide critical errors (e.g., during checkpoint)
	if (unlikely(f2fs_cp_error(F2FS_I_SB(inode))))
		return -EIO;

	// Check if the inode is immutable (cannot be changed)
	if (unlikely(IS_IMMUTABLE(inode)))
		return -EPERM;

	// Check if the inode is append-only and the operation attempts to change
	// mode, ownership, or timestamps. Size changes are handled separately.
	if (unlikely(IS_APPEND(inode) &&
			(attr->ia_valid & (ATTR_MODE | ATTR_UID |
				  ATTR_GID | ATTR_TIMES_SET))))
		return -EPERM;

	// Specific checks if the size attribute (ATTR_SIZE) is being changed
	if ((attr->ia_valid & ATTR_SIZE)) {
		// If compression is enabled but backend isn't ready, or if it's a device alias inode
		if (!f2fs_is_compress_backend_ready(inode) ||
				IS_DEVICE_ALIASING(inode))
			return -EOPNOTSUPP; // Operation not supported
		// If compression blocks were released and new size isn't cluster aligned
		if (is_inode_flag_set(inode, FI_COMPRESS_RELEASED) &&
			!IS_ALIGNED(attr->ia_size,
			F2FS_BLK_TO_BYTES(fi->i_cluster_size)))
			return -EINVAL; // Invalid argument
	}

	// Prepare for setting attributes (VFS helper).
	// This checks permissions, SUID/SGID clearing, etc.
	err = setattr_prepare(idmap, dentry, attr);
	if (err)
		return err;

	// Prepare for attribute changes if filesystem encryption is enabled
	err = fscrypt_prepare_setattr(dentry, attr);
	if (err)
		return err;

	// Prepare for attribute changes if fs-verity is enabled
	err = fsverity_prepare_setattr(dentry, attr);
	if (err)
		return err;

	// If the operation modifies attributes that affect quota
	if (is_quota_modification(idmap, inode, attr)) {
		err = f2fs_dquot_initialize(inode); // Initialize quota for the inode
		if (err)
			return err;
	}

	// If UID or GID needs to be updated
	if (i_uid_needs_update(idmap, attr, inode) ||
	    i_gid_needs_update(idmap, attr, inode)) {
		f2fs_lock_op(F2FS_I_SB(inode)); // Lock for filesystem operations
		// Transfer quota responsibility from old UID/GID to new
		err = dquot_transfer(idmap, inode, attr);
		if (err) {
			// If quota transfer fails, mark superblock for quota repair
			set_sbi_flag(F2FS_I_SB(inode),
					SBI_QUOTA_NEED_REPAIR);
			f2fs_unlock_op(F2FS_I_SB(inode));
			return err;
		}
		/*
		 * update uid/gid under lock_op(), so that dquot and inode can
		 * be updated atomically.
		 */
		i_uid_update(idmap, attr, inode); // Update inode's UID
		i_gid_update(idmap, attr, inode); // Update inode's GID
		f2fs_mark_inode_dirty_sync(inode, true); // Mark inode as dirty and sync
		f2fs_unlock_op(F2FS_I_SB(inode)); // Unlock filesystem operations
	}

	// If the ATTR_SIZE attribute is being changed (truncate or extend)
	if (attr->ia_valid & ATTR_SIZE) {
		loff_t old_size = i_size_read(inode); // Get current inode size

		// If new size is too large for inline data
		if (attr->ia_size > MAX_INLINE_DATA(inode)) {
			/*
			 * should convert inline inode before i_size_write to
			 * keep smaller than inline_data size with inline flag.
			 */
			// Convert from inline data storage to regular block-based storage
			err = f2fs_convert_inline_inode(inode);
			if (err)
				return err;
		}

		/*
		 * wait for inflight dio, blocks should be removed after
		 * IO completion.
		 */
		// If truncating to a smaller size, wait for any direct I/O to complete
		if (attr->ia_size < old_size)
			inode_dio_wait(inode);

		// Acquire write lock for garbage collection critical section
		f2fs_down_write(&fi->i_gc_rwsem[WRITE]);
		// Lock against page cache invalidation races 是和readhead进行并发保护的锁。
		filemap_invalidate_lock(inode->i_mapping);

		// VFS helper: sets inode->i_size and handles page cache for extension/truncation
		truncate_setsize(inode, attr->ia_size);

		// If truncating to a smaller or equal size, perform F2FS specific block truncation
		if (attr->ia_size <= old_size)
			err = f2fs_truncate(inode);
		/*
		 * do not trim all blocks after i_size if target size is
		 * larger than i_size. (f2fs_truncate handles this logic)
		 */
		filemap_invalidate_unlock(inode->i_mapping); // Release page cache invalidation lock
		f2fs_up_write(&fi->i_gc_rwsem[WRITE]); // Release GC lock
		if (err) // If f2fs_truncate failed
			return err;

		// Update timestamps and F2FS specific last disk size
		spin_lock(&fi->i_size_lock); // Lock for i_size related fields in f2fs_inode_info
		inode_set_mtime_to_ts(inode, inode_set_ctime_current(inode)); // Update mtime and ctime
		fi->last_disk_size = i_size_read(inode); // Store the new size
		spin_unlock(&fi->i_size_lock);
	}

	// VFS helper: copy other attributes (mode, uid, gid, times) to inode
	// This happens after size change, so i_uid_update/i_gid_update might have already done some work
	__setattr_copy(idmap, inode, attr);

	// If file mode is being changed
	if (attr->ia_valid & ATTR_MODE) {
		// Apply POSIX ACL changes based on the new mode
		err = posix_acl_chmod(idmap, dentry, f2fs_get_inode_mode(inode));

		// F2FS specific ACL mode handling
		if (is_inode_flag_set(inode, FI_ACL_MODE)) {
			if (!err) // If posix_acl_chmod was successful
				inode->i_mode = fi->i_acl_mode; // Restore mode from F2FS ACL cache
			clear_inode_flag(inode, FI_ACL_MODE); // Clear the flag
		}
	}

	/* file size may changed here */
	// Mark inode dirty again (could be redundant if size changed, but safe)
	f2fs_mark_inode_dirty_sync(inode, true); /*很重要的地方是inode会被标记为脏*/

	/* inode change will produce dirty node pages flushed by checkpoint */
	// Trigger filesystem balancing if needed (e.g., flush dirty data)
	f2fs_balance_fs(F2FS_I_SB(inode), true);

	return err; // Return final status (0 on success, or error from ACL/mode change)
}
```
