 **1. 函数调用链和数据结构解析**

**a) `vfs_fsync(struct file *file, int datasync)`**

```c
int vfs_fsync(struct file *file, int datasync)
{
	return vfs_fsync_range(file, 0, LLONG_MAX, datasync);
}
```

* **功能:**  VFS 层的 `fsync` 系统调用入口点。
* **参数:**
    * `file`:  指向要同步的文件的 `file` 结构体指针。
    * `datasync`:  标志，指示是否只执行 `fdatasync` 操作 (只同步数据和必要的元数据)。
* **操作:**  直接调用 `vfs_fsync_range` 函数，指定同步范围为整个文件 (从偏移量 0 到 `LLONG_MAX`)。

**b) `vfs_fsync_range(struct file *file, loff_t start, loff_t end, int datasync)`**

```c
int vfs_fsync_range(struct file *file, loff_t start, loff_t end, int datasync)
{
	struct inode *inode = file->f_mapping->host;

	if (!file->f_op->fsync)
		return -EINVAL;
	if (!datasync && (inode->i_state & I_DIRTY_TIME))
		mark_inode_dirty_sync(inode);
	return file->f_op->fsync(file, start, end, datasync);
}
```

* **功能:**  VFS 层的 `fsync` 范围同步函数。
* **参数:**
    * `file`:  指向要同步的文件的 `file` 结构体指针。
    * `start`, `end`:  同步的字节范围 (起始偏移量和结束偏移量，包含 `end` 偏移量)。
    * `datasync`:  `fdatasync` 标志。
* **操作:**
    1. **获取 inode:** 从 `file->f_mapping->host` 获取与文件关联的 `inode` 结构体。
    2. **检查 `fsync` 操作:** 检查 `file->f_op->fsync` 函数指针是否已设置 (文件系统是否提供了 `fsync` 实现)。 如果没有，返回 `-EINVAL` 错误。
    3. **标记 inode 为同步脏 (如果需要):**  如果 `datasync` 为 false (执行 `fsync`) 且 inode 标记了 `I_DIRTY_TIME` 标志 (表示 inode 的时间戳是脏的)，则调用 `mark_inode_dirty_sync(inode)` 将 inode 标记为需要同步写回。
    4. **调用文件系统特定的 `fsync` 函数:**  调用 `file->f_op->fsync(file, start, end, datasync)`，将实际的 `fsync` 操作委托给文件系统 (例如，F2FS) 提供的 `fsync` 实现。

**c) `f2fs_sync_file(struct file *file, loff_t start, loff_t end, int datasync)`**

* [f2fs_sync_file](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/file.c/f2fs_sync_file.md)

**d) `f2fs_do_sync_file(struct file *file, loff_t start, loff_t end, int datasync, bool atomic)`**

[f2fs_do_sync_file](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/file.c/f2fs_do_sync_file.md)

**e) `file_write_and_wait_range(struct file *file, loff_t lstart, loff_t lend)`**

[file_write_and_wait_range](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/filemap.c/file_write_and_wait_range.md)

**f) `__filemap_fdatawrite_range(struct address_space *mapping, loff_t start, loff_t end, int sync_mode)`**

[__filemap_fdatawrite_range](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/filemap.c/__filemap_fdatawrite_range.md)

**g) `filemap_fdatawrite_wbc(struct address_space *mapping, struct writeback_control *wbc)`**

[filemap_fdatawrite_wbc](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/filemap.c/filemap_fdatawrite_wbc.md)

**h) `do_writepages(struct address_space *mapping, struct writeback_control *wdo_writepagesbc)`**

[do_writepages](https://github.do_writepagescom/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/page_writeback.c/do_writepages.md)

**i) `f2fs_write_data_pages(struct address_space *mapping, struct writeback_control *wbc)`**

[f2fs_write_data_pages](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/data.c/f2fs_write_data_pages.md)

**j) `__f2fs_write_data_pages(struct address_space *mapping, struct writeback_control *wbc, enum iostat_type io_type)`**

[__f2fs_write_data_pages](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/data.c/__f2fs_write_data_pages.md)

