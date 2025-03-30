```c
int __filemap_fdatawrite_range(struct address_space *mapping, loff_t start,loff_t end, int sync_mode)
{
	struct writeback_control wbc = {
		.sync_mode = sync_mode,
		.nr_to_write = LONG_MAX,
		.range_start = start,
		.range_end = end,
	};

	return filemap_fdatawrite_wbc(mapping, &wbc);
}
```

* **功能:**  启动指定范围的 mapping 脏页的 writeback 操作。
* **参数:**
    * `mapping`:  要写回的 `address_space` 结构体。
    * `start`, `end`:  要写回的字节范围。
    * `sync_mode`:  同步模式 (`WB_SYNC_ALL`)。
* **操作:**
    1. **初始化 `writeback_control` 结构体 `wbc`:**  设置 `wbc` 的 `sync_mode`, `nr_to_write`, `range_start`, `range_end` 等字段，**配置 writeback 操作的参数**。
    2. **调用 `filemap_fdatawrite_wbc`:**  调用 `filemap_fdatawrite_wbc` 函数，**使用配置好的 `wbc` 结构体，进一步执行 writeback 操作**。