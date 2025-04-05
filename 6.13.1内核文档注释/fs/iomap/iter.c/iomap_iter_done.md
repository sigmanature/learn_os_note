```c
static inline void iomap_iter_done(struct iomap_iter *iter)
{
	WARN_ON_ONCE(iter->iomap.offset > iter->pos);
	WARN_ON_ONCE(iter->iomap.length == 0);
	WARN_ON_ONCE(iter->iomap.offset + iter->iomap.length <= iter->pos);
	WARN_ON_ONCE(iter->iomap.flags & IOMAP_F_STALE);

	trace_iomap_iter_dstmap(iter->inode, &iter->iomap);
	if (iter->srcmap.type != IOMAP_HOLE)
		trace_iomap_iter_srcmap(iter->inode, &iter->srcmap);
}
```

**功能:** `iomap_iter_done` 函数在每次 `iomap_iter` 成功调用 `ops->iomap_begin` 并获取到映射信息后被调用，用于进行一些完成处理，主要是完整性检查和跟踪。

**步骤分解:**

*   **`WARN_ON_ONCE(...)` (Line 3-6): 完整性检查**
    *   这些 `WARN_ON_ONCE` 宏用于进行一系列的完整性检查，如果条件为真，则会打印一次警告信息。
        *   `iter->iomap.offset > iter->pos`: 检查映射的起始偏移量是否大于当前文件偏移量，这应该是不正常的。
        *   `iter->iomap.length == 0`: 检查映射的长度是否为 0，如果为 0，可能表示没有获取到有效的映射。
        *   `iter->iomap.offset + iter->iomap.length <= iter->pos`: 检查映射的结束位置是否小于等于当前文件偏移量，这也不应该发生。
        *   `iter->iomap.flags & IOMAP_F_STALE`: 检查 `IOMAP_F_STALE` 标志是否被设置，在 `iomap_iter_done` 时，这个标志应该没有被设置。

*   **`trace_iomap_iter_dstmap(iter->inode, &iter->iomap);` (Line 8): 跟踪目标映射**
    *   记录一个跟踪点，用于记录目标文件系统块的映射信息 (`iter->iomap`)。

*   **`if (iter->srcmap.type != IOMAP_HOLE) trace_iomap_iter_srcmap(iter->inode, &iter->srcmap);` (Line 9-10): 跟踪源映射 (如果不是空洞)**
    *   如果 `iter->srcmap` 的类型不是 `IOMAP_HOLE` (表示不是空洞映射)，则记录一个跟踪点，用于记录源文件系统块的映射信息 (`iter->srcmap`)。 这通常在 reflink 操作等场景下使用。

**总结 `iomap_iter_done`:**

`iomap_iter_done` 函数主要用于在每次成功获取映射后进行一些断言检查，确保映射信息的有效性，并记录跟踪信息，方便调试和监控。 它本身不影响迭代的流程，主要是辅助性的功能。