```c
int f2fs_sync_file(struct file *file, loff_t start, loff_t end, int datasync)
{
	if (unlikely(f2fs_cp_error(F2FS_I_SB(file_inode(file)))))
		return -EIO;
	return f2fs_do_sync_file(file, start, end, datasync, false);
}
```

* **功能:**  F2FS 文件系统的 `fsync` 实现 (VFS 层调用的 `file->f_op->fsync` 函数指针指向这里)。
* **参数:**  与 `vfs_fsync_range` 相同。
* **操作:**
    1. **检查 CP 错误:**  检查 F2FS 的 checkpoint (CP) 机制是否发生错误 (`f2fs_cp_error`)。 如果有错误，返回 `-EIO` 错误 (I/O 错误)。
    2. **调用 `f2fs_do_sync_file`:**  调用 `f2fs_do_sync_file` 函数执行实际的同步操作。

```c
static int f2fs_do_sync_file(struct file *file, loff_t start, loff_t end,
						int datasync, bool atomic)
{
	// ... (函数代码) ...
}
```

* **功能:**  F2FS 实际执行文件同步的核心函数。
* **参数:**
    * `file`, `start`, `end`, `datasync`:  与之前相同。
    * `atomic`:  布尔标志，指示是否执行原子写操作 (用于某些特殊情况，例如 strict fsync 模式)。
* **关键数据结构: `struct writeback_control wbc`**

   ```c
   struct writeback_control wbc = {
       .sync_mode = WB_SYNC_ALL,
       .nr_to_write = LONG_MAX,
       .for_reclaim = 0,
   };
   ```
   * **`writeback_control` 结构体:**  这是一个非常重要的结构体，用于 **控制和配置 writeback 操作**。  它在 `fs/fs-writeback.c` 中定义，被多个文件系统和内核模块使用。
   * **`.sync_mode = WB_SYNC_ALL`:**  设置同步模式为 `WB_SYNC_ALL`，表示这是一个 **同步写回操作**，需要等待所有写回操作完成。
   * **`.nr_to_write = LONG_MAX`:**  设置要写入的页数为 `LONG_MAX`，表示 **没有页数限制**，尽可能多地写回脏页。
   * **`.for_reclaim = 0`:**  表示 **不是为了内存回收 (reclaim) 而进行的 writeback**。

* **函数 `f2fs_do_sync_file` 的主要操作 (简化流程):**
    1. **检查只读状态和 CP 错误:**  如果文件系统是只读的或 CP 发生错误，则直接返回。
    2. **设置 `FI_NEED_IPU` 标志 (In-Place Update):**  根据 `datasync` 标志或脏页数量，决定是否设置 `FI_NEED_IPU` 标志。  如果设置，可能会影响后续的写入方式 (例如，可能倾向于原地更新)。
    3. **调用 `file_write_and_wait_range`:**  **关键步骤！** 调用 `file_write_and_wait_range` 函数，**启动实际的数据页写回操作**，并等待写回完成。
    4. **清除 `FI_NEED_IPU` 标志:**  清除 `FI_NEED_IPU` 标志。
    5. **处理 inode 元数据同步:**  根据 inode 的脏状态和 `datasync` 标志，决定是否需要写回 inode 元数据 (`f2fs_write_inode`)。
    6. **检查是否需要 checkpoint:**  调用 `need_do_checkpoint` 检查是否需要执行 checkpoint 操作 (F2FS 的一致性机制)。
    7. **执行 checkpoint (如果需要):**  如果需要 checkpoint，调用 `f2fs_sync_fs` 执行文件系统级别的同步，包括写回所有脏的 node pages 和 meta pages。
    8. **同步 node pages:**  如果不需要 checkpoint，或者 checkpoint 完成后，调用 `f2fs_fsync_node_pages` 同步 inode 的 node pages (包含 inode 元数据和块指针等)。
    9. **等待 node pages 写回完成:**  如果不是原子写模式，调用 `f2fs_wait_on_node_pages_writeback` 等待 node pages 的写回操作完成。
    10. **清理 inode 标志和 ino entry:**  清除 inode 的 `FI_APPEND_WRITE`, `FI_UPDATE_WRITE` 等标志，并移除相关的 ino entry (用于跟踪 inode 的写入状态)。
    11. **执行 flush 操作 (如果需要):**  根据 `fsync_mode` 选项，可能调用 `f2fs_issue_flush` 发送 flush 命令到存储设备，确保数据持久化。
    12. **更新时间戳:**  调用 `f2fs_update_time` 更新文件系统的时间戳。