```c
int iomap_iter(struct iomap_iter *iter, const struct iomap_ops *ops)
{
	int ret;
	/*暂时忽略 如果文件系统提供了 `iomap_end` 操作，并且 `iter->iomap.length` 不为零（表示上一次迭代有映射信息），则在开始新的迭代之前，会调用 `ops->iomap_end` 进行一些清理工作，例如释放资源。*/
	if (iter->iomap.length && ops->iomap_end) {
		ret = ops->iomap_end(iter->inode, iter->pos, iomap_length(iter),
				iter->processed > 0 ? iter->processed : 0,
				iter->flags, &iter->iomap);
		if (ret < 0 && !iter->processed)
			return ret;
	}

	trace_iomap_iter(iter, ops, _RET_IP_);
	/*内核这么写其实很像迭代器中的next操作
	更新一步迭代器的状态,并且可以在里面做一些额外的事情。C++中一般就是使用++运算符更新迭代器的状态*/
	ret = iomap_iter_advance(iter);
	if (ret <= 0)
		return ret;

	ret = ops->iomap_begin(iter->inode, iter->pos, iter->len, iter->flags,
			       &iter->iomap, &iter->srcmap);
	if (ret < 0)
		return ret;
	iomap_iter_done(iter);
	return 1;
}
```

* **`ops->iomap_end` (可选)**:  如果文件系统提供了 `iomap_end` 操作，并且 `iter->iomap.length` 不为零（表示上一次迭代有映射信息），则在开始新的迭代之前，会调用 `ops->iomap_end` 进行一些清理工作，例如释放资源。
* **`iomap_iter_advance(iter)`**:  更新迭代器的状态，例如根据上一次迭代的处理长度 (`iter->processed`) 调整 `iter->pos` 和 `iter->len`，并重置 `iter->iomap` 和 `iter->srcmap`。
* **`ops->iomap_begin(iter->inode, iter->pos, iter->len, iter->flags, &iter->iomap, &iter->srcmap)`**: **核心步骤：获取映射信息**。调用文件系统提供的 `iomap_begin` 操作（对于 XFS 来说是 `xfs_read_iomap_begin`），请求获取从 `iter->pos` 开始，长度为 `iter->len` 的文件范围的映射信息。
    * `iter->inode`:  文件 inode。
    * `iter->pos`:  文件偏移量。
    * `iter->len`:  请求的长度。
    * `iter->flags`:  标志位。
    * `&iter->iomap`:  输出参数，用于接收文件块到磁盘块的映射信息。
    * `&iter->srcmap`:  输出参数，用于接收源映射信息（在 reflink 等场景下使用，这里可能为空）。
* **`iomap_iter_done(iter)`**:  记录和检查 `iter->iomap` 的状态，进行一些调试和断言检查。
* **返回值**:  `iomap_iter` 返回正值表示成功获取到映射信息，可以继续迭代；返回 0 或负值表示迭代结束或发生错误。