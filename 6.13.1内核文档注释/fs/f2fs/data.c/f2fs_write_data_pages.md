```c
static int f2fs_write_data_pages(struct address_space *mapping,
			    struct writeback_control *wbc)
{
	struct inode *inode = mapping->host;

	return __f2fs_write_data_pages(mapping, wbc,
			F2FS_I(inode)->cp_task == current ?
			FS_CP_DATA_IO : FS_DATA_IO);
}
```

* **功能:**  F2FS 文件系统用于写回数据页的 `writepages` 操作实现 (由 `do_writepages` 调用)。
* **参数:**
    * `mapping`:  要写回的 `address_space` 结构体。
    * `wbc`:  `writeback_control` 结构体。
* **操作:**  直接调用 `__f2fs_write_data_pages` 函数，并根据当前进程是否是 CP 任务 (Checkpoint 任务) 传递不同的 `io_type` 参数 (用于 I/O 统计)。