**相关函数**
* [iomap_iter](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/iomap/iter.c/iomap_iter.md)
* [iomap_write_iter](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/iomap/buffered_io.c/iomap_write_iter.md)
这个函数是 VFS/MM 层提供的通用 iomap 缓冲写入实现，被 `xfs_file_buffered_write` 调用。它负责迭代地调用文件系统提供的 `iomap_ops` 回调函数来获取文件块映射，并将用户数据拷贝到相应的页面缓存中。

```c
ssize_t // 函数返回写入的字节数或负的错误码
iomap_file_buffered_write(struct kiocb *iocb, // 内核 I/O 控制块
		struct iov_iter *i,             // 用户数据迭代器
		const struct iomap_ops *ops,    // 文件系统提供的 iomap 操作回调函数集合
		void *private)                  // 传递给回调函数的私有数据 (这里是 NULL)
{
	// 初始化 iomap 迭代器结构
	struct iomap_iter iter = {
		.inode		= iocb->ki_filp->f_mapping->host, // 目标 inode
		.pos		= iocb->ki_pos,                   // 起始写入位置
		.len		= iov_iter_count(i),              // 要写入的总长度
		.flags		= IOMAP_WRITE,                    // 操作标志：写入
		.private	= private,                        // 私有数据
	};
	ssize_t ret; // 用于存储返回值

	// 如果设置了非阻塞 I/O 标志
	if (iocb->ki_flags & IOCB_NOWAIT)
		iter.flags |= IOMAP_NOWAIT; // 将 NOWAIT 标志传递给 iomap 迭代器

	// 主循环：持续调用 iomap_iter 获取映射并处理数据
	// iomap_iter 会调用 ops->iomap_begin (即 xfs_buffered_write_iomap_begin)
	// ret > 0 表示 iomap_iter 成功返回了一个映射，其值为映射的长度
	while ((ret = iomap_iter(&iter, ops)) > 0)
		// iomap_write_iter 根据 iomap_iter 返回的映射 (存储在 iter.iomap 中)，
		// 将数据从用户迭代器 'i' 拷贝到对应的页面缓存页。
		// 它会更新 iter.pos 和 iter.len，并返回实际处理的字节数。
		iter.processed = iomap_write_iter(&iter, i);

	// 循环结束后 (iomap_iter 返回 0 或错误码)
	// 检查是否根本没有写入任何数据
	if (unlikely(iter.pos == iocb->ki_pos))
		// 如果 iter.pos 没有前进，说明一次数据都没写入，返回 iomap_iter 的错误码
		return ret;

	// 计算总共写入的字节数
	ret = iter.pos - iocb->ki_pos;
	// 更新 iocb 中的文件位置
	iocb->ki_pos = iter.pos;
	// 返回成功写入的字节数
	return ret;
}
```