```c
int do_writepages(struct address_space *mapping, struct writeback_control *wbc)
{
	int ret;
	struct bdi_writeback *wb;

	if (wbc->nr_to_write <= 0)
		return 0;
	wb = inode_to_wb_wbc(mapping->host, wbc);
	wb_bandwidth_estimate_start(wb);
	while (1) {
		if (mapping->a_ops->writepages) {
			ret = mapping->a_ops->writepages(mapping, wbc);
		} else if (mapping->a_ops->writepage) {
			ret = writeback_use_writepage(mapping, wbc);
		} else {
			/* deal with chardevs and other special files */
			ret = 0;
		}
		if (ret != -ENOMEM || wbc->sync_mode != WB_SYNC_ALL)
			break;

		/*
		 * Lacking an allocation context or the locality or writeback
		 * state of any of the inode's pages, throttle based on
		 * writeback activity on the local node. It's as good a
		 * guess as any.
		 */
		reclaim_throttle(NODE_DATA(numa_node_id()),
			VMSCAN_THROTTLE_WRITEBACK);
	}
	// ... (bandwidth update code) ...
	return ret;
}
```

* **功能:**  核心的 writepages 执行函数，负责调用文件系统提供的 `writepages` 或 `writepage` 操作，并处理 writeback 过程中的错误和重试。
* **参数:**
    * `mapping`:  要写回的 `address_space` 结构体。
    * `wbc`:  `writeback_control` 结构体。
* **操作:**
    1. **检查 `nr_to_write`:**  如果 `wbc->nr_to_write` 小于等于 0，表示不需要写回任何页，直接返回成功 (0)。
    2. **获取 `bdi_writeback` 结构体:**  调用 `inode_to_wb_wbc(mapping->host, wbc)` 获取与 inode 关联的 `bdi_writeback` 结构体，用于 writeback 统计和控制。
    3. **启动带宽估计:**  `wb_bandwidth_estimate_start(wb)` 启动 writeback 带宽估计。
    4. **循环调用 `writepages` 或 `writepage`:**  进入一个循环，在循环中：
        * **优先调用 `writepages`:**  如果 `mapping->a_ops->writepages` 函数指针已设置 (文件系统提供了批量写回页面的 `writepages` 操作)，则调用 `mapping->a_ops->writepages(mapping, wbc)`。  **对于 F2FS 来说，会调用 `f2fs_write_data_pages` (或类似的函数，取决于文件类型)。**
        * **备选调用 `writepage`:**  如果 `writepages` 未提供，但 `mapping->a_ops->writepage` 函数指针已设置 (文件系统提供了逐页写回的 `writepage` 操作)，则调用 `writeback_use_writepage(mapping, wbc)`，使用通用的逐页写回机制。
        * **处理特殊文件:**  如果 `writepages` 和 `writepage` 都未提供 (例如，对于字符设备等特殊文件)，则将 `ret` 设置为 0，表示无需写回。
        * **处理 `-ENOMEM` 错误和重试:**  如果 `writepages` 或 `writepage` 返回 `-ENOMEM` 错误 (表示内存不足)，并且同步模式是 `WB_SYNC_ALL` (同步写回)，则会进行 **内存回收节流 (reclaim_throttle)**，等待一段时间后 **重试** 写回操作。  如果不是 `-ENOMEM` 错误或不是同步写回，则跳出循环。
    5. **更新带宽估计:**  循环结束后，更新 writeback 带宽估计 (`wb_update_bandwidth`)。
    6. **返回 `writepages` 或 `writepage` 的返回值:**  返回最后一次 `writepages` 或 `writepage` 调用的返回值，表示 writeback 操作的结果。