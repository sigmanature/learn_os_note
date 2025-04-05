现在我们来深入分析 `xfs_bmapi_read` 函数，这是 XFS 文件系统中获取块映射关系的关键函数。

```c
typedef struct xfs_bmbt_irec
{
	xfs_fileoff_t	br_startoff;	/* starting file offset */
	xfs_fsblock_t	br_startblock;	/* starting block number */
	xfs_filblks_t	br_blockcount;	/* number of blocks */
	xfs_exntst_t	br_state;	/* extent state */
} xfs_bmbt_irec_t;
int
xfs_bmapi_read(
	struct xfs_inode	*ip,
	xfs_fileoff_t		bno,
	xfs_filblks_t		len,
	struct xfs_bmbt_irec	*mval,
	int			*nmap,
	uint32_t		flags)
{
	struct xfs_mount	*mp = ip->i_mount;
	int			whichfork = xfs_bmapi_whichfork(flags);
	struct xfs_ifork	*ifp = xfs_ifork_ptr(ip, whichfork);/*根据 fork 类型获取对应的 inode fork 结构 `ifp`。 `xfs_ifork` 结构包含了 extent 列表等信息，用于管理文件的数据或属性。*/
	struct xfs_bmbt_irec	got;
	/*声明一个 `xfs_bmbt_irec` 结构 `got`，用于在循环中存储每次从 extent 列表中查找到的 extent 信息。*/
	xfs_fileoff_t		obno;
	xfs_fileoff_t		end;
	struct xfs_iext_cursor	icur;
	int			error;
	bool			eof = false;
	int			n = 0;

	ASSERT(*nmap >= 1);
	ASSERT(!(flags & ~(XFS_BMAPI_ATTRFORK | XFS_BMAPI_ENTIRE)));
	xfs_assert_ilocked(ip, XFS_ILOCK_SHARED | XFS_ILOCK_EXCL);

	if (WARN_ON_ONCE(!ifp)) {
		xfs_bmap_mark_sick(ip, whichfork);
		return -EFSCORRUPTED;
	}

	if (XFS_IS_CORRUPT(mp, !xfs_ifork_has_extents(ifp)) ||
	    XFS_TEST_ERROR(false, mp, XFS_ERRTAG_BMAPIFORMAT)) {
		xfs_bmap_mark_sick(ip, whichfork);
		return -EFSCORRUPTED;
	}

	if (xfs_is_shutdown(mp))
		return -EIO;

	XFS_STATS_INC(mp, xs_blk_mapr);

	error = xfs_iread_extents(NULL, ip, whichfork);
	if (error)
		return error;

	if (!xfs_iext_lookup_extent(ip, ifp, bno, &icur, &got))
		eof = true;/*这一步已经会对传进来的bno做一次查询了,结果存储在got里*/
	end = bno + len;/*len也是从函数参数传*/
	obno = bno;/*保存从函数参数传进来的*/

	while (bno < end && n < *nmap) {
		/* Reading past eof, act as though there's a hole up to end. */
		if (eof)
			got.br_startoff = end;
		if (got.br_startoff > bno) {
			/* Reading in a hole.  */
			mval->br_startoff = bno;/*确实如xfs_bmbt_irec所说
			br_startoff一定是从文件偏移转来的块号*/
			mval->br_startblock = HOLESTARTBLOCK;/*这个值就是个特殊值 是-2*/
			mval->br_blockcount =
				XFS_FILBLKS_MIN(len, got.br_startoff - bno);
				/*这里只能是空洞数量了 公式是取剩余长度和从bno表示的块号到从extent查询得到的起始块号之间距离的最小值实际上意味着整个这条extent记录全部表示空洞*/
			mval->br_state = XFS_EXT_NORM;
			bno += mval->br_blockcount;/*迭代的更新逻辑 让bno跳过当前记录的块数量*/
			len -= mval->br_blockcount;/*迭代的更新逻辑 让剩余长度(块数为单位)减去当前记录的块数量*/
			mval++;/*指针迭代,看来实际上它是个数组*/
			n++;/*记录数量加1 只是这条记录全是空洞了*/
			continue;
		}

		/* set up the extent map to return. */
		xfs_bmapi_trim_map(mval, &got, &bno, len, obno, end, n, flags);/*这应该就是没文件空洞的时候,从got里获取一个记录了*/
		xfs_bmapi_update_map(&mval, &got, &bno, len, obno, end, &n, flags);

		/* If we're done, stop now. */
		if (bno >= end || n >= *nmap)
			break;

		/* Else go on to the next record. */
		if (!xfs_iext_next_extent(ifp, &icur, &got))
			eof = true;
	}
	*nmap = n;
	return 0;/*相当于把mval这个irec数组给填满*/
}
```

**功能:** `xfs_bmapi_read` 函数是 XFS 文件系统中用于读取文件块到文件系统块映射关系的核心函数。它根据给定的文件块号和长度，查找并返回对应的块映射信息。

**步骤分解:**

*   **`struct xfs_mount *mp = ip->i_mount;` (Line 8): 获取 XFS mount 结构**
    *   从 `xfs_inode` 结构 `ip` 中获取 XFS mount 结构 `mp`。

*   **`int whichfork = xfs_bmapi_whichfork(flags);` (Line 9): 确定 fork 类型**
    *   `xfs_bmapi_whichfork(flags)` 函数根据 `flags` 确定要操作的 fork 类型 (数据 fork 或属性 fork)。 对于普通文件 I/O，通常是数据 fork。

*   **`struct xfs_ifork *ifp = xfs_ifork_ptr(ip, whichfork);` (Line 10): 获取 inode fork 结构**
    *   `xfs_ifork_ptr(ip, whichfork)` 函数根据 fork 类型获取对应的 inode fork 结构 `ifp`。 `xfs_ifork` 结构包含了 extent 列表等信息，用于管理文件的数据或属性。

*   **`struct xfs_bmbt_irec got;` (Line 11): 声明 `xfs_bmbt_irec` 结构 `got`**
    *   声明一个 `xfs_bmbt_irec` 结构 `got`，用于在循环中存储每次从 extent 列表中查找到的 extent 信息。

*   **`xfs_fileoff_t obno;` (Line 12): 声明原始块号 `obno`**
    *   声明 `obno` 变量，用于保存原始的起始块号 `bno`，在循环中使用。

*   **`xfs_fileoff_t end;` (Line 13): 声明结束块号 `end`**
    *   声明 `end` 变量，用于保存 I/O 操作的结束块号 (`bno + len`)。

*   **`struct xfs_iext_cursor icur;` (Line 14): 声明 extent 列表游标 `icur`**
    *   声明 `xfs_iext_cursor` 结构 `icur`，用于在 inode 的 extent 列表中进行查找和迭代。

*   **`int error;` (Line 15): 声明错误码 `error`**
    *   声明 `error` 变量，用于存储错误码。

*   **`bool eof = false;` (Line 16): 初始化 `eof` 标志**
    *   `eof = false`: 初始化 `eof` 标志为 `false`，用于指示是否到达文件末尾 (extent 列表的末尾)。

*   **`int n = 0;` (Line 17): 初始化映射计数器 `n`**
    *   `n = 0`: 初始化映射计数器 `n` 为 0，用于记录已返回的映射记录数量。

*   **`ASSERT(*nmap >= 1);` (Line 19): 断言检查 `nmap`**
    *   断言检查 `*nmap` (期望返回的映射记录数量) 是否大于等于 1。

*   **`ASSERT(!(flags & ~(XFS_BMAPI_ATTRFORK | XFS_BMAPI_ENTIRE)));` (Line 20): 断言检查标志位**
    *   断言检查 `flags` 中是否只设置了 `XFS_BMAPI_ATTRFORK` 或 `XFS_BMAPI_ENTIRE` 标志，或者没有设置任何标志。  这限制了 `xfs_bmapi_read` 函数可以处理的标志位类型。

*   **`xfs_assert_ilocked(ip, XFS_ILOCK_SHARED | XFS_ILOCK_EXCL);` (Line 21): 断言检查 inode 锁**
    *   `xfs_assert_ilocked(...)` 断言检查 inode `ip` 是否持有共享锁 (`XFS_ILOCK_SHARED`) 或排他锁 (`XFS_ILOCK_EXCL`)。 `xfs_bmapi_read` 函数需要在 inode 锁的保护下调用。

*   **`if (WARN_ON_ONCE(!ifp)) { ... }` (Line 23-25): 警告并处理无效 ifork**
    *   `WARN_ON_ONCE(!ifp)`: 如果 `ifp` 为空 (表示 inode fork 不存在)，则打印一次警告信息。
    *   `xfs_bmap_mark_sick(ip, whichfork);`: 标记 bmap 为 "sick" (不健康)，可能表示文件系统元数据损坏。
    *   `return -EFSCORRUPTED;`: 返回 `-EFSCORRUPTED` 错误，表示文件系统元数据损坏。

*   **`if (XFS_IS_CORRUPT(mp, !xfs_ifork_has_extents(ifp)) || ...` (Line 27-29): 检查文件系统损坏状态**
    *   检查文件系统是否处于损坏状态。
        *   `XFS_IS_CORRUPT(mp, !xfs_ifork_has_extents(ifp))`: 检查文件系统是否标记为损坏，并且 inode fork 是否没有 extent。
        *   `XFS_TEST_ERROR(false, mp, XFS_ERRTAG_BMAPIFORMAT)`:  测试是否模拟 `XFS_ERRTAG_BMAPIFORMAT` 错误 (用于错误注入测试)。
    *   如果文件系统损坏，则执行与上面类似的操作：标记 bmap 为 "sick" 并返回 `-EFSCORRUPTED` 错误。

*   **`if (xfs_is_shutdown(mp)) return -EIO;` (Line 31): 检查文件系统是否关闭**
    *   如果 XFS 文件系统已经关闭，则返回 `-EIO` 错误。

*   **`XFS_STATS_INC(mp, xs_blk_mapr);` (Line 33): 增加块映射读取统计**
    *   `XFS_STATS_INC(mp, xs_blk_mapr)` 宏增加 XFS 文件系统的块映射读取统计计数器 `xs_blk_mapr`。

*   **`error = xfs_iread_extents(NULL, ip, whichfork);` (Line 35): 读取 extent 列表**
    *   `xfs_iread_extents(NULL, ip, whichfork)` 函数从磁盘读取 inode 的 extent 列表到内存中。 extent 列表描述了文件的数据块在磁盘上的分布。

*   **`if (error) return error;` (Line 36): 检查读取 extent 列表错误**
    *   如果 `xfs_iread_extents` 返回错误，则直接返回错误。

*   **`if (!xfs_iext_lookup_extent(ip, ifp, bno, &icur, &got)) eof = true;` (Line 38-39): 查找起始 extent**
    *   `xfs_iext_lookup_extent(...)` 函数在 inode fork `ifp` 的 extent 列表中查找包含文件块号 `bno` 的 extent。
        *   **参数:**
            *   `ip`: XFS inode 结构。
            *   `ifp`: inode fork 结构。
            *   `bno`:  起始文件块号。
            *   `&icur`: 指向 extent 列表游标 `icur` 的指针，用于定位查找位置。
            *   `&got`: 指向 `xfs_bmbt_irec` 结构 `got` 的指针，用于存储查找到的 extent 信息。
        *   **返回值:** 如果找到 extent，返回 true，并将 extent 信息存储在 `got` 中； 如果没有找到 (例如，`bno` 超出文件范围)，返回 false。
    *   **`if (!xfs_iext_lookup_extent(...)) eof = true;`**: 如果 `xfs_iext_lookup_extent` 返回 false (未找到 extent)，则设置 `eof = true`，表示起始块号 `bno` 已经超出文件范围。

*   **`end = bno + len;` (Line 40): 计算结束块号 `end`**
    *   计算 I/O 操作的结束块号 `end`。

*   **`obno = bno;` (Line 41): 保存原始起始块号**
    *   保存原始的起始块号 `bno` 到 `obno` 变量中，在循环中使用。

*   **`while (bno < end && n < *nmap) { ... }` (Line 43-80): 主循环，迭代处理 extent**
    *   主循环，只要当前块号 `bno` 小于结束块号 `end` 并且已返回的映射记录数量 `n` 小于期望的数量 `*nmap`，循环就继续。

    *   **`/* Reading past eof, act as though there's a hole up to end. */ if (eof) got.br_startoff = end;` (Line 44-45): 处理读取超出 EOF 情况**
        *   如果 `eof` 标志为 true (表示起始块号 `bno` 已经超出文件范围)，则将 `got.br_startoff` 设置为 `end`。 这样在后续处理中，会将超出文件范围的部分视为一个空洞 (hole)。

    *   **`if (got.br_startoff > bno) { ... }` (Line 46-57): 处理空洞 (hole)**
        *   如果当前 extent 的起始文件块号 `got.br_startoff` 大于当前块号 `bno`，表示当前位置是一个空洞 (hole)。
        *   **`mval->br_startoff = bno;`**: 设置当前映射记录 `mval` 的起始文件块号为 `bno`。
        *   **`mval->br_startblock = HOLESTARTBLOCK;`**: 设置 `mval` 的起始设备块号为 `HOLESTARTBLOCK`，表示这是一个空洞。
        *   **`mval->br_blockcount = XFS_FILBLKS_MIN(len, got.br_startoff - bno);`**: 计算空洞的块数量，取剩余长度 `len` 和当前块号 `bno` 到 extent 起始位置 `got.br_startoff` 之间的较小值。
        *   **`mval->br_state = XFS_EXT_NORM;`**: 设置 `mval` 的状态为 `XFS_EXT_NORM` (正常 extent 状态，即使是空洞也用这个状态)。
        *   **`bno += mval->br_blockcount;`**: 更新当前块号 `bno`，加上空洞的块数量。
        *   **`len -= mval->br_blockcount;`**: 更新剩余长度 `len`，减去空洞的块数量。
        *   **`mval++; n++; continue;`**: 指向下一个映射记录 `mval`，增加映射记录计数器 `n`，并继续下一次循环迭代。

    *   **`/* set up the extent map to return. */ ...` (Line 60-61): 设置 extent 映射信息**
        *   **`xfs_bmapi_trim_map(mval, &got, &bno, len, obno, end, n, flags);`**: `xfs_bmapi_trim_map` 函数用于裁剪 extent 映射信息，确保返回的 extent 不超出请求的范围。
        *   **`xfs_bmapi_update_map(&mval, &got, &bno, len, obno, end, &n, flags);`**: `xfs_bmapi_update_map` 函数用于更新当前映射记录 `mval` 的信息，例如起始块号、设备块号、块数量等，并更新循环控制变量 `bno`、`len` 和 `n`。

    *   **`/* If we're done, stop now. */ if (bno >= end || n >= *nmap) break;` (Line 64-65): 检查是否完成**
        *   如果当前块号 `bno` 已经达到或超过结束块号 `end`，或者已返回的映射记录数量 `n` 已经达到期望的数量 `*nmap`，则跳出循环，表示映射获取完成。

    *   **`/* Else go on to the next record. */ if (!xfs_iext_next_extent(ifp, &icur, &got)) eof = true;` (Line 67-68): 获取下一个 extent**
        *   如果循环没有结束，则调用 `xfs_iext_next_extent(...)` 函数获取 inode fork `ifp` 的 extent 列表中游标 `icur` 指向的下一个 extent。
            *   **参数:**
                *   `ifp`: inode fork 结构。
                *   `&icur`: 指向 extent 列表游标 `icur` 的指针，函数会更新游标指向下一个 extent。
                *   `&got`: 指向 `xfs_bmbt_irec` 结构 `got` 的指针，用于存储下一个 extent 的信息.
            *   **返回值:** 如果成功获取到下一个 extent，返回 true，并将 extent 信息存储在 `got` 中； 如果到达 extent 列表末尾，返回 false。
        *   **`if (!xfs_iext_next_extent(...)) eof = true;`**: 如果 `xfs_iext_next_extent` 返回 false (到达 extent 列表末尾)，则设置 `eof = true`。

*   **`*nmap = n;` (Line 79): 更新实际返回的映射记录数量**
    *   将实际返回的映射记录数量 `n` 赋值给 `*nmap`，以便调用者知道实际返回了多少个映射记录。

*   **`return 0;` (Line 80): 返回成功**
    *   返回 0，表示 `xfs_bmapi_read` 函数执行成功。

**总结 `xfs_bmapi_read`:**

`xfs_bmapi_read` 函数是 XFS 文件系统中块映射获取的核心实现。它通过以下步骤完成映射查找：

1.  **初始化:** 获取 inode fork、初始化游标、计数器等。
2.  **读取 extent 列表:** 从磁盘读取 inode 的 extent 列表到内存。
3.  **查找起始 extent:** 在 extent 列表中查找包含起始文件块号的 extent。
4.  **迭代处理 extent:**  在一个循环中，迭代处理 extent 列表，直到处理完请求的范围或者达到期望的映射记录数量。
    *   **处理空洞:**  如果当前位置是空洞，则生成一个描述空洞的映射记录。
    *   **裁剪和更新映射:**  调用 `xfs_bmapi_trim_map` 和 `xfs_bmapi_update_map` 函数来裁剪和更新 extent 映射信息。
    *   **获取下一个 extent:**  获取 extent 列表中的下一个 extent，继续处理。
5.  **返回结果:**  返回实际返回的映射记录数量和成功状态。

`xfs_bmapi_read` 函数的核心逻辑是遍历 inode 的 extent 列表，根据请求的文件范围，生成一系列描述文件块到文件系统块映射关系的 `xfs_bmbt_irec` 结构。 它能够处理文件中的 extent 和空洞，并返回连续的映射记录，以便上层 I/O 代码进行数据传输。