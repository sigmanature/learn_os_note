**函数关系调用图**
```mermaid
graph TD
E[f2fs_do_sync_file]-->b
subgraph b[file_write_and_wait_range]
A{如果mapping中有记录的脏页}--是-->B["__filemap_fdatawrite_range<br>(mapping, lstart, lend,<br>WB_SYNC_ALL)"]-->C{如果错误不是EIO}--是-->D["__filemap_fdatawait_range<br>(mapping, lstart, lend)"]-->a
end 
subgraph a[__filemap_fdatawrite_range]
1["初始化wbc,<br>同步模式为传入的模式<br>,<br>
.range_start = start,<br>
.range_end = end,"]-->2["filemap_fdatawrite_wbc<br>(mapping, &wbc)"]-->c
end
subgraph c[filemap_fdatawrite_wbc]
3[wbc_attach_fdatawrite_inode]-->4[do_writepages]-->5["循环调用a_ops->writepages"]-->6[f2fs_write_data_pages]
end
click E https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/file.c/f2fs_do_sync_file.md
click 6 https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/data.c/f2fs_write_data_pages.md
```
```c
int file_write_and_wait_range(struct file *file, loff_t lstart, loff_t lend)
{
	int err = 0, err2;
	struct address_space *mapping = file->f_mapping;

	if (lend < lstart)
		return 0;

	if (mapping_needs_writeback(mapping)) {
		err = __filemap_fdatawrite_range(mapping, lstart, lend,WB_SYNC_ALL);
		/* See comment of filemap_write_and_wait() */
		if (err != -EIO)
			__filemap_fdatawait_range(mapping, lstart, lend);
	}
	err2 = file_check_and_advance_wb_err(file);
	if (!err)
		err = err2;
	return err;
}
```

* **功能:**  写回并等待指定文件范围的数据页。
* **参数:**
    * `file`:  指向文件的 `file` 结构体指针。
    * `lstart`, `lend`:  要写回的字节范围。
* **操作:**
    1. **检查范围:**  如果 `lend < lstart`，表示范围无效，直接返回成功 (0)。
    2. **检查是否需要 writeback:**  调用 `mapping_needs_writeback(mapping)` 检查 `address_space` 是否需要 writeback (例如，是否有脏页)。
    3. **调用 `__filemap_fdatawrite_range` 启动写回:**  如果需要 writeback，调用 `__filemap_fdatawrite_range` 函数，**启动针对指定范围的数据页的写回操作**，同步模式为 `WB_SYNC_ALL`。
    4. **调用 `__filemap_fdatawait_range` 等待写回完成:**  如果 `__filemap_fdatawrite_range` 没有返回 `-EIO` 错误 (表示没有发生严重的 I/O 错误)，则调用 `__filemap_fdatawait_range` 函数，**等待之前启动的写回操作完成**。
    5. **检查并更新 writeback 错误:**  调用 `file_check_and_advance_wb_err` 检查是否有 writeback 过程中产生的错误，并更新 `file` 结构体中的错误状态。
    6. **返回错误码:**  返回写回过程中遇到的错误码。