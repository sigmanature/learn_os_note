```c
int filemap_fdatawrite_wbc(struct address_space *mapping,
			   struct writeback_control *wbc)
{
	int ret;

	if (!mapping_can_writeback(mapping) ||
	    !mapping_tagged(mapping, PAGECACHE_TAG_DIRTY))
		return 0;

	wbc_attach_fdatawrite_inode(wbc, mapping->host);
	ret = do_writepages(mapping, wbc);
	wbc_detach_inode(wbc);
	return ret;
}
```

* **功能:**  使用提供的 `writeback_control` 结构体，调用 `writepages` 操作来启动 mapping 的脏页写回。
* **参数:**
    * `mapping`:  要写回的 `address_space` 结构体。
    * `wbc`:  `writeback_control` 结构体，包含 writeback 配置信息。
* **操作:**
    1. **检查是否可以 writeback:**  调用 `mapping_can_writeback(mapping)` 检查 `address_space` 是否支持 writeback。
    2. **检查是否有脏页:**  调用 `mapping_tagged(mapping, PAGECACHE_TAG_DIRTY)` 检查 `address_space` 是否有标记为 `PAGECACHE_TAG_DIRTY` 的脏页。  如果没有，则无需写回，直接返回成功 (0)。
    3. **关联 inode 到 `wbc`:**  调用 `wbc_attach_fdatawrite_inode(wbc, mapping->host)` 将 inode 关联到 `writeback_control` 结构体，用于 writeback 记账和 cgroup 管理。
    4. **调用 `do_writepages` 执行写回:**  **关键步骤！** 调用 `do_writepages(mapping, wbc)` 函数，**真正执行数据页的写回操作**。  `do_writepages` 会进一步调用文件系统 (例如，F2FS) 提供的 `writepages` 或 `writepage` address_space operations。
    5. **解除 inode 与 `wbc` 的关联:**  调用 `wbc_detach_inode(wbc)` 解除 inode 与 `writeback_control` 的关联。
    6. **返回 `do_writepages` 的返回值:**  返回 `do_writepages` 函数的返回值，表示 writeback 操作的结果。