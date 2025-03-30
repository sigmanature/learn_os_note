```c
static int __f2fs_write_data_pages(struct address_space *mapping,
				   struct writeback_control *wbc,
				   enum iostat_type io_type)
{
	// ... (函数代码) ...
}
```

* **功能:**  `f2fs_write_data_pages` 的实际实现函数，负责收集脏页并调用 `f2fs_write_cache_pages` 进行写回。
* **参数:**
    * `mapping`:  要写回的 `address_space` 结构体。
    * `wbc`:  `writeback_control` 结构体.
    * `io_type`:  I/O 类型 (用于 I/O 统计)。
* **操作 (简化流程):**
    1. **检查 `writepage` 操作:**  检查 `mapping->a_ops->writepage` 是否已实现。 如果没有，直接返回成功 (0)。
    2. **跳过写回 (某些情况):**  在某些条件下 (例如，POR 恢复中、inode 脏页数少于阈值且内存充足、文件碎片整理准备阶段等)，可能会跳过写回操作。
    3. **追踪 `f2fs_writepages` 事件:**  `trace_f2fs_writepages` 用于性能追踪。
    4. **处理同步/异步写回请求:**  根据 `wbc->sync_mode` 和全局的 writeback 请求计数器 (`sbi->wb_sync_req[DATA]`)，进行同步或异步写回处理。  避免同步和异步写回混合导致 I/O 分裂。
    5. **获取 `writepages` 互斥锁 (如果需要):**  如果需要序列化 I/O 操作 (`__should_serialize_io`)，则获取 `sbi->writepages` 互斥锁，保证同一时刻只有一个进程执行 `writepages` 操作。
    6. **启动 blk plug:**  `blk_start_plug(&plug)` 启动 blk plug 机制，用于合并和优化块设备 I/O 请求。
    7. **调用 `f2fs_write_cache_pages` 执行写回:**  **关键步骤！** 调用 `f2fs_write_cache_pages(mapping, wbc, io_type)` 函数，**实际收集脏页并提交写回 I/O 请求**。  **这是我们接下来要重点分析的函数。**
    8. **完成 blk plug:**  `blk_finish_plug(&plug)` 完成 blk plug，提交合并后的 I/O 请求。
    9. **释放 `writepages` 互斥锁 (如果获取了):**  释放之前获取的 `sbi->writepages` 互斥锁。
    10. **更新 inode 脏 inode 列表:**  `f2fs_remove_dirty_inode(inode)` 将 inode 从脏 inode 列表中移除 (表示正在进行写回)。
    11. **返回 `f2fs_write_cache_pages` 的返回值:**  返回 `f2fs_write_cache_pages` 函数的返回值，表示写回操作的结果。