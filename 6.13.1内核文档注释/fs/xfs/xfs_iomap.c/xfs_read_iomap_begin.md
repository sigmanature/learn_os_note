**相关函数**
*	[xfs_bmapi_read](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/xfs/xfs_iomap.c/xfs_bmapi_read.md)
```c
static int
xfs_read_iomap_begin(
	struct inode		*inode,
	loff_t			offset,
	loff_t			length,
	unsigned		flags,
	struct iomap		*iomap,
	struct iomap		*srcmap)
{
	struct xfs_inode	*ip = XFS_I(inode);
	struct xfs_mount	*mp = ip->i_mount;
	struct xfs_bmbt_irec	imap;
	/*offset就是iomap_iter中的偏移 length也是iomap_iter中的length*/
	/*声明一个 `xfs_bmbt_irec` 结构 `imap`，用于存储从 `xfs_bmapi_read` 获取的块映射记录。 `xfs_bmbt_irec` 是 XFS 文件系统内部表示块映射信息的结构。*/
	xfs_fileoff_t		offset_fsb = XFS_B_TO_FSBT(mp, offset);
	/*将字节偏移量 `offset` 转换为文件系统块偏移量 `offset_fsb`。
	这个应该是起始的呃文件系统块偏移了*/
	xfs_fileoff_t		end_fsb = xfs_iomap_end_fsb(mp, offset, length);
	/*计算 I/O 操作范围的结束文件系统块偏移量*/
	int			nimaps = 1, error = 0;
    /*nimaps是最重要的变量没有之一
    它控制xfs_bmapi_read最多返回一条记录*/
	bool			shared = false;
	unsigned int		lockmode = XFS_ILOCK_SHARED;
	u64			seq;

	ASSERT(!(flags & (IOMAP_WRITE | IOMAP_ZERO)));

	if (xfs_is_shutdown(mp))
		return -EIO;

	error = xfs_ilock_for_iomap(ip, flags, &lockmode);
	if (error)
		return error;
	error = xfs_bmapi_read(ip, offset_fsb, end_fsb - offset_fsb, &imap,
			       &nimaps, 0);/*给定起始映射的文件块以及映射数量 返回imap映射信息
				   以及记录的映射信息数量*/
	if (!error && ((flags & IOMAP_REPORT) || IS_DAX(inode)))
		error = xfs_reflink_trim_around_shared(ip, &imap, &shared);
	seq = xfs_iomap_inode_sequence(ip, shared ? IOMAP_F_SHARED : 0);
	xfs_iunlock(ip, lockmode);

	if (error)
		return error;
	trace_xfs_iomap_found(ip, offset, length, XFS_DATA_FORK, &imap);
	return xfs_bmbt_to_iomap(ip, iomap, &imap, flags,
				 shared ? IOMAP_F_SHARED : 0, seq);/*这一步是要将块映射信息转为iomap结构体里存储的信息*/
}
```

**功能:** `xfs_read_iomap_begin` 是 XFS 文件系统为读取操作提供的 `iomap_begin` 实现。它负责获取指定文件范围在 XFS 文件系统上的块映射信息。

**步骤分解:**

*   **`struct xfs_inode *ip = XFS_I(inode);` (Line 8): 获取 XFS inode 结构**
    *   `XFS_I(inode)` 宏将通用的 `inode` 结构转换为 XFS 特定的 `xfs_inode` 结构 `ip`。

*   **`struct xfs_mount *mp = ip->i_mount;` (Line 9): 获取 XFS mount 结构**
    *   从 `xfs_inode` 结构 `ip` 中获取 XFS mount 结构 `mp`，包含了文件系统挂载点的信息。

*   **`struct xfs_bmbt_irec imap;` (Line 10): 声明 `xfs_bmbt_irec` 结构**
    *   声明一个 `xfs_bmbt_irec` 结构 `imap`，用于存储从 `xfs_bmapi_read` 获取的块映射记录。 `xfs_bmbt_irec` 是 XFS 文件系统内部表示块映射信息的结构。

*   **`xfs_fileoff_t offset_fsb = XFS_B_TO_FSBT(mp, offset);` (Line 11): 转换为文件系统块偏移量**
    *   `XFS_B_TO_FSBT(mp, offset)` 宏将字节偏移量 `offset` 转换为文件系统块偏移量 `offset_fsb`。

*   **`xfs_fileoff_t end_fsb = xfs_iomap_end_fsb(mp, offset, length);` (Line 12): 计算结束文件系统块偏移量**
    *   `xfs_iomap_end_fsb(mp, offset, length)` 函数计算 I/O 操作范围的结束文件系统块偏移量 `end_fsb`。

*   **`int nimaps = 1, error = 0;` (Line 13): 初始化变量**
    *   `nimaps = 1`: 初始化 `nimaps` 为 1，表示期望获取的映射记录数量为 1。 `xfs_bmapi_read` 函数会更新这个值，表示实际返回的映射记录数量。
    *   `error = 0`: 初始化错误码 `error` 为 0。

*   **`bool shared = false;` (Line 14): 初始化 `shared` 标志**
    *   `shared = false`: 初始化 `shared` 标志为 `false`，用于指示是否是共享 extent (用于 reflink)。

*   **`unsigned int lockmode = XFS_ILOCK_SHARED;` (Line 15): 初始化锁模式**
    *   `lockmode = XFS_ILOCK_SHARED`: 初始化锁模式为共享锁 `XFS_ILOCK_SHARED`，因为是读取操作。

*   **`u64 seq;` (Line 16): 声明序列号变量**
    *   声明 `seq` 变量，用于存储 inode 的序列号，用于并发控制。

*   **`ASSERT(!(flags & (IOMAP_WRITE | IOMAP_ZERO)));` (Line 18): 断言检查标志**
    *   断言检查 `flags` 中是否设置了 `IOMAP_WRITE` 或 `IOMAP_ZERO` 标志。 `xfs_read_iomap_begin` 是为读取操作设计的，不应该处理写入或 zeroing 操作。

*   **`if (xfs_is_shutdown(mp)) return -EIO;` (Line 20): 检查文件系统是否关闭**
    *   如果 XFS 文件系统已经关闭 (`xfs_is_shutdown(mp)` 为真)，则返回 `-EIO` 错误。

*   **`error = xfs_ilock_for_iomap(ip, flags, &lockmode);` (Line 22): 获取 inode 锁**
    *   `xfs_ilock_for_iomap(ip, flags, &lockmode)` 函数获取 inode 锁。根据 `flags` 和 `lockmode` 参数，获取适当的锁 (共享锁或排他锁)。

*   **`error = xfs_bmapi_read(ip, offset_fsb, end_fsb - offset_fsb, &imap, &nimaps, 0);` (Line 24-25): 调用 `xfs_bmapi_read` 获取块映射**
    *   **`xfs_bmapi_read(...)`**:  调用 `xfs_bmapi_read` 函数，这是获取 XFS 文件块到文件系统块映射关系的核心函数。 我们稍后会详细分析 `xfs_bmapi_read`。
        *   **参数:**
            *   `ip`: XFS inode 结构。
            *   `offset_fsb`: 起始文件系统块偏移量。
            *   `end_fsb - offset_fsb`:  需要映射的文件系统块数量。
            *   `&imap`: 指向 `xfs_bmbt_irec` 结构的指针，用于存储返回的映射信息。
            *   `&nimaps`: 指向 `nimaps` 变量的指针，表示期望和实际返回的映射记录数量。
            *   `0`:  标志位，这里为 0。

*   **`if (!error && ((flags & IOMAP_REPORT) || IS_DAX(inode))) ...` (Line 26-27): 处理 reflink 和 DAX**
    *   如果 `xfs_bmapi_read` 没有返回错误，并且设置了 `IOMAP_REPORT` 标志或者 inode 是 DAX (Direct Access) 设备，则调用 `xfs_reflink_trim_around_shared` 函数。 这部分代码可能与 reflink (copy-on-write) 和 DAX 设备的优化有关，用于处理共享 extent 的裁剪。

*   **`seq = xfs_iomap_inode_sequence(ip, shared ? IOMAP_F_SHARED : 0);` (Line 28): 获取 inode 序列号**
    *   `xfs_iomap_inode_sequence(ip, shared ? IOMAP_F_SHARED : 0)` 函数获取 inode 的序列号。序列号用于并发控制，`IOMAP_F_SHARED` 标志根据 `shared` 变量的值设置。

*   **`xfs_iunlock(ip, lockmode);` (Line 29): 释放 inode 锁**
    *   `xfs_iunlock(ip, lockmode)` 函数释放之前获取的 inode 锁。

*   **`if (error) return error;` (Line 31): 检查错误**
    *   如果之前
**第 31-35 行:**  错误检查、跟踪和最终映射转换。

*   **`if (error) return error;` (Line 31): 再次检查错误**
    *   在获取 inode 锁、调用 `xfs_bmapi_read` 以及可能的 `xfs_reflink_trim_around_shared` 操作之后，再次检查 `error` 变量。如果 `error` 非零，表示在之前的步骤中发生了错误，直接返回该错误码，终止映射过程。

*   **`trace_xfs_iomap_found(ip, offset, length, XFS_DATA_FORK, &imap);` (Line 32): 跟踪找到的 iomap**
    *   `trace_xfs_iomap_found(...)` 是一个跟踪点，用于记录成功找到的 iomap 信息。
        *   **参数:**
            *   `ip`: XFS inode 结构。
            *   `offset`:  原始的字节偏移量。
            *   `length`:  原始的字节长度。
            *   `XFS_DATA_FORK`:  指示这是数据 fork 的映射 (相对于属性 fork)。
            *   `&imap`: 指向 `xfs_bmbt_irec` 结构的指针，包含了 `xfs_bmapi_read` 返回的块映射信息。

*   **`return xfs_bmbt_to_iomap(ip, iomap, &imap, flags, shared ? IOMAP_F_SHARED : 0, seq);` (Line 33-35): 转换 XFS 块映射到通用 iomap 结构并返回**
    *   **`xfs_bmbt_to_iomap(...)`**:  调用 `xfs_bmbt_to_iomap` 函数，将 XFS 特定的块映射信息 (`imap`，类型为 `xfs_bmbt_irec`) 转换为通用的 `iomap` 结构 (`iomap`)。 `iomap` 结构是 iomap 子系统通用的映射描述结构，可以被上层 I/O 代码使用。
        *   **参数:**
            *   `ip`: XFS inode 结构。
            *   `iomap`: 指向目标 `iomap` 结构的指针，用于存储转换后的通用映射信息. 这是 `xfs_read_iomap_begin` 函数的主要输出之一。
            *   `&imap`: 指向源 `xfs_bmbt_irec` 结构的指针，包含了 XFS 特定的块映射信息。
            *   `flags`:  原始的 I/O 操作标志。
            *   `shared ? IOMAP_F_SHARED : 0`:  根据 `shared` 标志决定是否设置 `IOMAP_F_SHARED` 标志。
            *   `seq`:  inode 的序列号。
        *   **返回值:** `xfs_bmbt_to_iomap` 函数的返回值会直接作为 `xfs_read_iomap_begin` 的返回值返回。 通常情况下，成功返回 0，失败返回负的错误码。

**总结 `xfs_read_iomap_begin`:**

`xfs_read_iomap_begin` 函数是 XFS 文件系统为读取操作提供的 `iomap_begin` 的具体实现。它的主要职责是：

1.  **准备参数:**  将字节偏移量和长度转换为文件系统块偏移量和数量。
2.  **获取 inode 锁:**  为了数据一致性，获取 inode 的共享锁。
3.  **调用 `xfs_bmapi_read`:**  调用核心的 `xfs_bmapi_read` 函数，从 XFS 文件系统的元数据中查找指定范围的块映射信息，并将结果存储在 `xfs_bmbt_irec` 结构中。
4.  **处理 reflink/DAX (可选):**  根据标志位和设备类型，进行可能的 reflink 和 DAX 相关的优化处理。
5.  **获取 inode 序列号:**  获取 inode 的序列号，用于并发控制。
6.  **释放 inode 锁:**  释放之前获取的 inode 锁。
7.  **转换映射格式:**  调用 `xfs_bmbt_to_iomap` 函数，将 XFS 特定的块映射信息转换为通用的 `iomap` 结构。
8.  **返回结果:**  返回操作结果，成功通常返回 0，失败返回错误码。

`xfs_read_iomap_begin` 的核心在于调用 `xfs_bmapi_read` 来获取 XFS 文件系统的块映射信息，并将结果转换为 iomap 子系统可以理解的通用格式。
