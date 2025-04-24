linux truncate系统调用会走到具体文件系统inode->ops中的setattr函数调用。从名字上看不出来,但确实这个函数执行了实际的truncate操作。
**相关函数**
* [page_cache_ra_order](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/readahead.c/page_cache_ra_order.md)
* [truncate_setsize](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/readahead.c/truncate_setsize.md)
```C
int f2fs_setattr(struct mnt_idmap *idmap, struct dentry *dentry,
		 struct iattr *attr)
{
	struct inode *inode = d_inode(dentry);
	struct f2fs_inode_info *fi = F2FS_I(inode);
	int err;

	if (unlikely(f2fs_cp_error(F2FS_I_SB(inode))))
		return -EIO;

	if (unlikely(IS_IMMUTABLE(inode)))
		return -EPERM;

	if (unlikely(IS_APPEND(inode) &&
			(attr->ia_valid & (ATTR_MODE | ATTR_UID |
				  ATTR_GID | ATTR_TIMES_SET))))
		return -EPERM;

	if ((attr->ia_valid & ATTR_SIZE)) {
		if (!f2fs_is_compress_backend_ready(inode) ||
				IS_DEVICE_ALIASING(inode))
			return -EOPNOTSUPP;
		if (is_inode_flag_set(inode, FI_COMPRESS_RELEASED) &&
			!IS_ALIGNED(attr->ia_size,
			F2FS_BLK_TO_BYTES(fi->i_cluster_size)))
			return -EINVAL;
	}

	err = setattr_prepare(idmap, dentry, attr);
	if (err)
		return err;

	err = fscrypt_prepare_setattr(dentry, attr);
	if (err)
		return err;

	err = fsverity_prepare_setattr(dentry, attr);
	if (err)
		return err;

	if (is_quota_modification(idmap, inode, attr)) {
		err = f2fs_dquot_initialize(inode);
		if (err)
			return err;
	}
	if (i_uid_needs_update(idmap, attr, inode) ||
	    i_gid_needs_update(idmap, attr, inode)) {
		f2fs_lock_op(F2FS_I_SB(inode));
		err = dquot_transfer(idmap, inode, attr);
		if (err) {
			set_sbi_flag(F2FS_I_SB(inode),
					SBI_QUOTA_NEED_REPAIR);
			f2fs_unlock_op(F2FS_I_SB(inode));
			return err;
		}
		/*
		 * update uid/gid under lock_op(), so that dquot and inode can
		 * be updated atomically.
		 */
		i_uid_update(idmap, attr, inode);
		i_gid_update(idmap, attr, inode);
		f2fs_mark_inode_dirty_sync(inode, true);
		f2fs_unlock_op(F2FS_I_SB(inode));
	}

	if (attr->ia_valid & ATTR_SIZE) {
		loff_t old_size = i_size_read(inode);

		if (attr->ia_size > MAX_INLINE_DATA(inode)) {
			/*
			 * should convert inline inode before i_size_write to
			 * keep smaller than inline_data size with inline flag.
			 */
			err = f2fs_convert_inline_inode(inode);
			if (err)
				return err;
		}

		/*
		 * wait for inflight dio, blocks should be removed after
		 * IO completion.
		 */
		if (attr->ia_size < old_size)
			inode_dio_wait(inode);

		f2fs_down_write(&fi->i_gc_rwsem[WRITE]);/*先上了gc的读写锁*/
		filemap_invalidate_lock(inode->i_mapping);
        /*最核心的并发逻辑,独占形式上i_mapping的invalidate锁。和共享模式上锁的page_cache_ra_order形成互斥地访问*/

		truncate_setsize(inode, attr->ia_size);/*核心函数,真正执行truncate 页面中page/folio的操作*/

		if (attr->ia_size <= old_size)
			err = f2fs_truncate(inode);/*核心函数,真正执行truncate数据块的操作*/
		/*
		 * do not trim all blocks after i_size if target size is
		 * larger than i_size.
		 */
		filemap_invalidate_unlock(inode->i_mapping);
		f2fs_up_write(&fi->i_gc_rwsem[WRITE]);
		if (err)
			return err;

		spin_lock(&fi->i_size_lock);
		inode_set_mtime_to_ts(inode, inode_set_ctime_current(inode));
		fi->last_disk_size = i_size_read(inode);
		spin_unlock(&fi->i_size_lock);
	}

	__setattr_copy(idmap, inode, attr);

	if (attr->ia_valid & ATTR_MODE) {
		err = posix_acl_chmod(idmap, dentry, f2fs_get_inode_mode(inode));

		if (is_inode_flag_set(inode, FI_ACL_MODE)) {
			if (!err)
				inode->i_mode = fi->i_acl_mode;
			clear_inode_flag(inode, FI_ACL_MODE);
		}
	}

	/* file size may changed here */
	f2fs_mark_inode_dirty_sync(inode, true);/*注意元数据会被标记为脏*/

	/* inode change will produce dirty node pages flushed by checkpoint */
	f2fs_balance_fs(F2FS_I_SB(inode), true);

	return err;
}
```