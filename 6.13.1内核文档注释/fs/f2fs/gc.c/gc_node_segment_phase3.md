  好的，我们来深入剖析 `gc_node_segment` 的 Phase 2 阶段，详细分析相关函数，解答你关于并发控制、数据搬移以及 `old_blkaddr`/`new_blkaddr` 的疑问。

**1. Phase 2 相关函数代码详解**

我们逐个分析 Phase 2 中用到的关键函数，理解其实现细节。

*   **`check_valid_map(struct f2fs_sb_info *sbi, unsigned int segno, int offset)`**

```c
static int check_valid_map(struct f2fs_sb_info *sbi,
				unsigned int segno, int offset)
{
	struct sit_info *sit_i = SIT_I(sbi);
	struct seg_entry *sentry;
	int ret;

	down_read(&sit_i->sentry_lock);
	sentry = get_seg_entry(sbi, segno);
	ret = f2fs_test_bit(offset, sentry->cur_valid_map);
	up_read(&sit_i->sentry_lock);
	return ret;
}
```

*   **功能:** `check_valid_map` 函数用于 **检查给定 Segment (`segno`) 和偏移量 (`offset`) 对应的 Block 是否有效**。  它通过查询 SIT (Segment Information Table) 中的有效性位图来实现。

*   **参数:**
    *   `struct f2fs_sb_info *sbi`: 文件系统超级块信息。
    *   `unsigned int segno`: 要检查的 Segment 号。
    *   `int offset`: 要检查的 Block 在 Segment 内的偏移量。

*   **获取 `sit_info` 和 `seg_entry`:**
    ```c
    struct sit_info *sit_i = SIT_I(sbi);
    struct seg_entry *sentry;
    ```
    *   `struct sit_info *sit_i = SIT_I(sbi);`:  获取 `sit_info` 结构体指针 `sit_i`，包含了 SIT 的管理信息。
    *   `struct seg_entry *sentry;`:  声明 `struct seg_entry` 指针 `sentry`，用于指向 Segment 的 SIT 条目。

*   **获取 `sentry_lock` 读锁:**  `down_read(&sit_i->sentry_lock);`:  **获取 `sit_i->sentry_lock` 读信号量**。  `sentry_lock` 是一个读写信号量，用于 **保护 SIT 缓存 (`sit_i->sentries`) 的并发访问**。  `down_read` 获取读锁，允许多个读者同时访问 SIT 缓存，但会互斥写者。

*   **获取 `seg_entry`:**  `sentry = get_seg_entry(sbi, segno);`:  调用 `get_seg_entry` 函数 **获取 Segment 号为 `segno` 的 `seg_entry` 结构体指针**。  `seg_entry` 结构体缓存了 Segment 的 SIT 信息，包括有效性位图。

*   **检查有效性位图:**  `ret = f2fs_test_bit(offset, sentry->cur_valid_map);`:  调用 `f2fs_test_bit` 函数 **检查 `sentry->cur_valid_map` 位图的第 `offset` 位是否被设置**。  `cur_valid_map` 是 `seg_entry` 结构体中的有效性位图，用于记录 Segment 内每个 Block 的有效性状态。  如果第 `offset` 位被设置，表示该 Block 有效，否则无效。

*   **释放 `sentry_lock` 读锁:**  `up_read(&sit_i->sentry_lock);`:  **释放 `sit_i->sentry_lock` 读信号量**。  在完成 SIT 缓存访问后，必须释放读锁，允许其他线程访问 SIT 缓存。

*   **返回值:**  `return ret;`:  返回 `f2fs_test_bit` 的返回值 `ret`。  `ret` 为非零值 (通常为 1) 表示 Block 有效，`ret` 为 0 表示 Block 无效。

*   **`get_seg_entry(struct f2fs_sb_info *sbi, unsigned int segno)`**

```c
static inline struct seg_entry *get_seg_entry(struct f2fs_sb_info *sbi,
						unsigned int segno)
{
	struct sit_info *sit_i = SIT_I(sbi);
	return &sit_i->sentries[segno];
}
```

*   **功能:** `get_seg_entry` 函数用于 **获取给定 Segment 号 (`segno`) 对应的 `seg_entry` 结构体指针**。  `seg_entry` 结构体是 SIT 缓存中的条目，缓存了 Segment 的 SIT 信息。

*   **参数:**
    *   `struct f2fs_sb_info *sbi`: 文件系统超级块信息。
    *   `unsigned int segno`: 要获取 `seg_entry` 的 Segment 号。

*   **获取 `sit_info` 和 `sentries` 数组:**  `struct sit_info *sit_i = SIT_I(sbi);`:  获取 `sit_info` 结构体指针 `sit_i`。  `sit_i->sentries` 是 SIT 缓存，是一个 `seg_entry` 结构体数组，缓存了所有 Segment 的 SIT 信息。

*   **返回 `seg_entry` 指针:**  `return &sit_i->sentries[segno];`:  **直接返回 `sit_i->sentries` 数组中索引为 `segno` 的元素的地址**。  `&sit_i->sentries[segno]` 就是指向 Segment 号为 `segno` 的 `seg_entry` 结构体的指针。  **`get_seg_entry` 函数只是一个简单的内联函数，用于快速访问 SIT 缓存中的 `seg_entry` 条目。**

*   **`f2fs_get_node_info(struct f2fs_sb_info *sbi, nid_t nid, struct node_info *ni, bool checkpoint_context)`**

```c
int f2fs_get_node_info(struct f2fs_sb_info *sbi, nid_t nid,
				struct node_info *ni, bool checkpoint_context)
{
	struct f2fs_nm_info *nm_i = NM_I(sbi);
	struct curseg_info *curseg = CURSEG_I(sbi, CURSEG_HOT_DATA);
	struct f2fs_journal *journal = curseg->journal;
	nid_t start_nid = START_NID(nid);
	struct f2fs_nat_block *nat_blk;
	struct page *page = NULL;
	struct f2fs_nat_entry ne;
	struct nat_entry *e;
	pgoff_t index;
	block_t blkaddr;
	int i;

	ni->nid = nid;
retry:
	/* Check nat cache */
	f2fs_down_read(&nm_i->nat_tree_lock);
	e = __lookup_nat_cache(nm_i, nid);
	if (e) {
		ni->ino = nat_get_ino(e);
		ni->blk_addr = nat_get_blkaddr(e);
		ni->version = nat_get_version(e);
		f2fs_up_read(&nm_i->nat_tree_lock);
		return 0;
	}

	/*
	 * Check current segment summary by trying to grab journal_rwsem first.
	 * This sem is on the critical path on the checkpoint requiring the above
	 * nat_tree_lock. Therefore, we should retry, if we failed to grab here
	 * while not bothering checkpoint.
	 */
	if (!f2fs_rwsem_is_locked(&sbi->cp_global_sem) || checkpoint_context) {
		down_read(&curseg->journal_rwsem);
	} else if (f2fs_rwsem_is_contended(&nm_i->nat_tree_lock) ||
				!down_read_trylock(&curseg->journal_rwsem)) {
		f2fs_up_read(&nm_i->nat_tree_lock);
		goto retry;
	}

	i = f2fs_lookup_journal_in_cursum(journal, NAT_JOURNAL, nid, 0);
	if (i >= 0) {
		ne = nat_in_journal(journal, i);
		node_info_from_raw_nat(ni, &ne);
	}
	up_read(&curseg->journal_rwsem);
	if (i >= 0) {
		f2fs_up_read(&nm_i->nat_tree_lock);
		goto cache;
	}

	/* Fill node_info from nat page */
	index = current_nat_addr(sbi, nid);
	f2fs_up_read(&nm_i->nat_tree_lock);

	page = f2fs_get_meta_page(sbi, index);
	if (IS_ERR(page))
		return PTR_ERR(page);

	nat_blk = (struct f2fs_nat_block *)page_address(page);
	ne = nat_blk->entries[nid - start_nid];
	node_info_from_raw_nat(ni, &ne);
	f2fs_put_page(page, 1);
cache:
	blkaddr = le32_to_cpu(ne.block_addr);
	if (__is_valid_data_blkaddr(blkaddr) &&
		!f2fs_is_valid_blkaddr(sbi, blkaddr, DATA_GENERIC_ENHANCE))
		return -EFAULT;

	/* cache nat entry */
	cache_nat_entry(sbi, nid, &ne);
	return 0;
}
```

*   **功能:** `f2fs_get_node_info` 函数用于 **获取给定 NID (`nid`) 的节点信息 (`struct node_info`)**。  节点信息包括 inode 号、块地址和版本号。  `f2fs_get_node_info` 函数会 **依次尝试从 NAT 缓存、Journal 日志和磁盘 NAT 页面中查找节点信息**。

*   **参数:**
    *   `struct f2fs_sb_info *sbi`: 文件系统超级块信息。
    *   `nid_t nid`: 要获取节点信息的 NID。
    *   `struct node_info *ni`:  输出参数，用于存储获取到的节点信息。
    *   `bool checkpoint_context`:  指示是否在 checkpoint 上下文中调用。  checkpoint 上下文可能影响锁的获取策略。

*   **变量初始化:**  函数开始处声明和初始化了多个局部变量，用于后续的 NAT 信息查找和处理。

*   **`retry:` 标签和 NAT 缓存查找:**
    ```c
	retry:
	/* Check nat cache */
	f2fs_down_read(&nm_i->nat_tree_lock);
	e = __lookup_nat_cache(nm_i, nid);
	if (e) {
		ni->ino = nat_get_ino(e);
		ni->blk_addr = nat_get_blkaddr(e);
		ni->version = nat_get_version(e);
		f2fs_up_read(&nm_i->nat_tree_lock);
		return 0;
	}
    ```
    *   **`retry:` 标签:**  用于在特定情况下重试 NAT 信息查找。
    *   **获取 `nat_tree_lock` 读锁:**  `f2fs_down_read(&nm_i->nat_tree_lock);`:  **获取 `nm_i->nat_tree_lock` 读信号量**。  `nat_tree_lock` 用于 **保护 NAT 缓存 (通常是一个 R-tree 或类似的结构) 的并发访问**。
    *   **查找 NAT 缓存:**  `e = __lookup_nat_cache(nm_i, nid);`:  调用 `__lookup_nat_cache` 函数 **在 NAT 缓存中查找 NID 对应的 `nat_entry` 结构体**。  `nat_entry` 结构体是 NAT 缓存的条目，缓存了 NAT 信息。
    *   **缓存命中处理:**  `if (e) { ... }`:  如果 `__lookup_nat_cache` 返回非 `NULL` 值 (表示缓存命中)，则从 `nat_entry` 中 **提取 inode 号、块地址和版本号**，填充到 `ni` 结构体中，并 **释放 `nat_tree_lock` 读锁**，然后 **返回 0 (成功)**。  **优先从 NAT 缓存中查找，可以提高性能。**

*   **Journal 日志查找:**
    ```c
    /*
     * Check current segment summary by trying to grab journal_rwsem first.
     * This sem is on the critical path on the checkpoint requiring the above
     * nat_tree_lock. Therefore, we should retry, if we failed to grab here
     * while not bothering checkpoint.
     */
    if (!f2fs_rwsem_is_locked(&sbi->cp_global_sem) || checkpoint_context) {
        down_read(&curseg->journal_rwsem);
    } else if (f2fs_rwsem_is_contended(&nm_i->nat_tree_lock) ||
                !down_read_trylock(&curseg->journal_rwsem)) {
        f2fs_up_read(&nm_i->nat_tree_lock);
        goto retry;
    }

    i = f2fs_lookup_journal_in_cursum(journal, NAT_JOURNAL, nid, 0);
    if (i >= 0) {
        ne = nat_in_journal(journal, i);
        node_info_from_raw_nat(ni, &ne);
    }
    up_read(&curseg->journal_rwsem);
    if (i >= 0) {
        f2fs_up_read(&nm_i->nat_tree_lock);
        goto cache;
    }
    ```
    *   **获取 `journal_rwsem` 读锁 (条件性):**  这段代码尝试获取 `curseg->journal_rwsem` 读信号量，用于 **保护 Journal 日志的并发访问**。  获取 `journal_rwsem` 的条件比较复杂，涉及到与 checkpoint 锁 (`sbi->cp_global_sem`) 和 `nat_tree_lock` 的交互，以及 `checkpoint_context` 参数。  **目的是在保证并发安全的前提下，尽可能避免锁竞争和死锁。**  如果获取 `journal_rwsem` 失败，可能会跳转到 `retry` 标签，重新开始 NAT 信息查找流程。
    *   **查找 Journal 日志:**  `i = f2fs_lookup_journal_in_cursum(journal, NAT_JOURNAL, nid, 0);`:  调用 `f2fs_lookup_journal_in_cursum` 函数 **在 Journal 日志中查找 NID 对应的 NAT 日志条目**。  `NAT_JOURNAL` 表示查找 NAT 日志类型。
    *   **Journal 命中处理:**  `if (i >= 0) { ... }`:  如果 `f2fs_lookup_journal_in_cursum` 返回非负值 (表示 Journal 命中)，则从 Journal 日志条目中 **提取 NAT 信息**，填充到 `ne` 结构体中，并调用 `node_info_from_raw_nat` 函数将 `ne` 中的信息转换为 `node_info` 格式，填充到 `ni` 结构体中。  然后 **释放 `journal_rwsem` 读锁**，并跳转到 `cache` 标签，进行后续的缓存操作。  **从 Journal 日志中查找 NAT 信息是第二优先级，在 NAT 缓存未命中时尝试。**

*   **磁盘 NAT 页面查找:**
    ```c
    /* Fill node_info from nat page */
    index = current_nat_addr(sbi, nid);
    f2fs_up_read(&nm_i->nat_tree_lock);

    page = f2fs_get_meta_page(sbi, index);
    if (IS_ERR(page))
        return PTR_ERR(page);

    nat_blk = (struct f2fs_nat_block *)page_address(page);
    ne = nat_blk->entries[nid - start_nid];
    node_info_from_raw_nat(ni, &ne);
    f2fs_put_page(page, 1);
	cache:
    // ... (后续缓存和验证操作) ...
    ```
    *   **计算 NAT 页面索引:**  `index = current_nat_addr(sbi, nid);`:  调用 `current_nat_addr` 函数 **计算 NID 对应的 NAT 页面索引 (物理块地址)**。
    *   **释放 `nat_tree_lock` 读锁:**  `f2fs_up_read(&nm_i->nat_tree_lock);`:  **释放 `nat_tree_lock` 读信号量**。  在访问磁盘 NAT 页面之前，需要释放 NAT 缓存的读锁，避免锁竞争。
    *   **获取 NAT 页面:**  `page = f2fs_get_meta_page(sbi, index);`:  调用 `f2fs_get_meta_page` 函数 **从磁盘读取 NAT 页面**。  `index` 参数是 NAT 页面的物理块地址。
    *   **错误处理:**  `if (IS_ERR(page)) return PTR_ERR(page);`:  如果 `f2fs_get_meta_page` 返回错误指针，则直接返回错误。
    *   **从 NAT 页面提取 NAT 条目:**  `nat_blk = (struct f2fs_nat_block *)page_address(page); ne = nat_blk->entries[nid - start_nid];`:  将 NAT 页面转换为 `f2fs_nat_block` 结构体指针，然后 **从 `nat_blk->entries` 数组中提取 NID 对应的 `f2fs_nat_entry` 结构体**。  `nid - start_nid` 计算了 NID 在 NAT 块内的偏移量。
    *   **转换为 `node_info`:**  `node_info_from_raw_nat(ni, &ne);`:  调用 `node_info_from_raw_nat` 函数将 `f2fs_nat_entry` 结构体 `ne` 中的信息转换为 `node_info` 格式，填充到 `ni` 结构体中。
    *   **释放 NAT 页面:**  `f2fs_put_page(page, 1);`:  **释放 NAT 页面的引用计数**。  在从 NAT 页面提取信息后，可以释放页面。

*   **`cache:` 标签和后续操作:**
    ```c
	cache:
	blkaddr = le32_to_cpu(ne.block_addr);
	if (__is_valid_data_blkaddr(blkaddr) &&
		!f2fs_is_valid_blkaddr(sbi, blkaddr, DATA_GENERIC_ENHANCE))
		return -EFAULT;

	/* cache nat entry */
	cache_nat_entry(sbi, nid, &ne);
	return 0;
    ```
    *   **`cache:` 标签:**  Journal 命中或磁盘 NAT 页面查找成功后，跳转到 `cache` 标签。
    *   **块地址有效性验证:**  `blkaddr = le32_to_cpu(ne.block_addr); if (__is_valid_data_blkaddr(blkaddr) && !f2fs_is_valid_blkaddr(sbi, blkaddr, DATA_GENERIC_ENHANCE)) return -EFAULT;`:  **验证从 NAT 中获取的块地址 `blkaddr` 的有效性**。  确保块地址在有效的数据块地址范围内。  如果块地址无效，则返回 `-EFAULT` 错误。
    *   **缓存 NAT 条目:**  `cache_nat_entry(sbi, nid, &ne);`:  调用 `cache_nat_entry` 函数 **将从磁盘或 Journal 获取的 `f2fs_nat_entry` 结构体 `ne` 缓存到 NAT 缓存中**，以便下次快速访问。
    *   **返回值:**  `return 0;`:  函数执行成功，返回 0。

*   **总结 `f2fs_get_node_info`:**  `f2fs_get_node_info` 函数实现了 **多级 NAT 信息查找机制**，依次尝试从 **NAT 缓存、Journal 日志和磁盘 NAT 页面** 中获取节点信息，并最终将获取到的 NAT 条目缓存到 NAT 缓存中，提高了 NAT 信息访问的效率和性能。  函数内部使用了 **读信号量 (`nat_tree_lock`, `journal_rwsem`)** 来保护 NAT 缓存和 Journal 日志的并发访问，保证了并发安全性。

*   **`START_NID(nid)` 宏**

```c
#define	START_NID(nid) (((nid) / NAT_ENTRY_PER_BLOCK) * NAT_ENTRY_PER_BLOCK)
```

*   **功能:** `START_NID(nid)` 宏计算 **给定 NID (`nid`) 对应的 NAT 块的起始 NID**。  由于每个 NAT 块可以存储 `NAT_ENTRY_PER_BLOCK` 个 NAT 条目，因此，连续 `NAT_ENTRY_PER_BLOCK` 个 NID 共享同一个 NAT 块。  `START_NID` 宏用于计算这组 NID 的起始 NID。

*   **计算方式:**  使用 `nid` 除以 `NAT_ENTRY_PER_BLOCK`，然后将结果乘以 `NAT_ENTRY_PER_BLOCK`。  这相当于 **将 `nid` 向下取整到 `NAT_ENTRY_PER_BLOCK` 的整数倍**。  例如，如果 `NAT_ENTRY_PER_BLOCK` 为 512，`nid` 为 1050，则 `START_NID(nid)` 为 (1050 / 512) * 512 = 512。  **这意味着 NID 1050 属于 NID 范围 512-1023，该范围的起始 NID 为 512。**

*   **`struct sit_info` 结构体**

```c
struct sit_info {
	block_t sit_base_addr;		/* start block address of SIT area */
	block_t sit_blocks;		/* # of blocks used by SIT area */
	block_t written_valid_blocks;	/* # of valid blocks in main area */
	char *bitmap;			/* all bitmaps pointer */
	char *sit_bitmap;		/* SIT bitmap pointer */
#ifdef CONFIG_F2FS_CHECK_FS
	char *sit_bitmap_mir;		/* SIT bitmap mirror */

	/* bitmap of segments to be ignored by GC in case of errors */
	unsigned long *invalid_segmap;
#endif
	unsigned int bitmap_size;	/* SIT bitmap size */

	unsigned long *tmp_map;			/* bitmap for temporal use */
	unsigned long *dirty_sentries_bitmap;	/* bitmap for dirty sentries */
	unsigned int dirty_sentries;		/* # of dirty sentries */
	unsigned int sents_per_block;		/* # of SIT entries per block */
	struct rw_semaphore sentry_lock;	/* to protect SIT cache */
	struct seg_entry *sentries;		/* SIT segment-level cache */
	struct sec_entry *sec_entries;		/* SIT section-level cache */

	/* for cost-benefit algorithm in cleaning procedure */
	unsigned long long elapsed_time;	/* elapsed time after mount */
	unsigned long long mounted_time;	/* mount time */
	unsigned long long min_mtime;		/* min. modification time */
	unsigned long long max_mtime;		/* max. modification time */
	unsigned long long dirty_min_mtime;	/* rerange candidates in GC_AT */
	unsigned long long dirty_max_mtime;	/* rerange candidates in GC_AT */

	unsigned int last_victim[MAX_GC_POLICY]; /* last victim segment # */
};
```

*   **功能:** `struct sit_info` 结构体定义了 **Segment Information Table (SIT) 的管理信息**。  SIT 负责维护 Segment 的状态信息，例如有效块数量、有效性位图、GC 状态等。

*   **字段 (部分重要字段):**
    *   `block_t sit_base_addr;`:  **SIT 区域的起始块地址**。
    *   `block_t sit_blocks;`:  **SIT 区域占用的块数量**。
    *   `char *sit_bitmap;`:  **SIT 位图指针**。  SIT 位图可能用于记录 Segment 的整体状态 (例如，是否空闲，是否正在 GC 等)。
    *   `unsigned int bitmap_size;`:  **SIT 位图的大小**。
    *   `struct rw_semaphore sentry_lock;`:  **`sentry_lock` 读写信号量**。  用于保护 SIT 缓存 (`sentries`) 的并发访问。
    *   `struct seg_entry *sentries;`:  **SIT 缓存 (Segment-level cache)**。  `sentries` 是一个 `seg_entry` 结构体数组，缓存了所有 Segment 的 SIT 信息。
    *   `struct sec_entry *sec_entries;`:  **SIT section-level cache**。  `sec_entries` 可能用于缓存 Section 级别的 SIT 信息 (Section 是 Segment 的分组)。

*   **`f2fs_move_node_page(struct page *node_page, int gc_type)` (再次分析，重点关注 `__write_node_page`)**

我们之前已经分析过 `f2fs_move_node_page` 函数，这里再次回顾，并重点关注其中调用的 `__write_node_page` 函数 (虽然 `__write_node_page` 的定义没有直接提供，但我们可以根据函数名和参数推测其功能)。

```c
int f2fs_move_node_page(struct page *node_page, int gc_type)
{
	int err = 0;

	if (gc_type == FG_GC) {
		struct writeback_control wbc = { ... };

		f2fs_wait_on_page_writeback(node_page, NODE, true, true);

		set_page_dirty(node_page);

		if (!clear_page_dirty_for_io(node_page)) {
			err = -EAGAIN;
			goto out_page;
		}

		if (__write_node_page(node_page, false, NULL,
					&wbc, false, FS_GC_NODE_IO, NULL)) { // 关键函数
			err = -EAGAIN;
			unlock_page(node_page);
		}
		goto release_page;
	} else {
		/* set page dirty and write it */
		if (!folio_test_writeback(page_folio(node_page)))
			set_page_dirty(node_page);
	}
out_page:
	unlock_page(node_page);
release_page:
	f2fs_put_page(node_page, 0);
	return err;
}
```

*   **`__write_node_page(node_page, false, NULL, &wbc, false, FS_GC_NODE_IO, NULL)` (推测功能):**  根据函数名和参数，可以推测 `__write_node_page` 函数的功能是 **将节点页面 `node_page` 写回到存储设备**。

    *   `node_page`:  要写回的节点页面。
    *   `false`:  可能是同步/异步写回标志，`false` 可能表示异步写回，但根据 `wbc.sync_mode = WB_SYNC_ALL` 的设置，这里实际是同步写回。
    *   `NULL, NULL`:  可能是用于传递额外参数的指针，这里为 `NULL`，表示没有额外参数。
    *   `&wbc`:  `writeback_control` 结构体指针，用于控制写回行为 (同步写回)。
    *   `false`:  可能是其他标志位，含义未知。
    *   `FS_GC_NODE_IO`:  IO 类型标志，表示 GC 引起的节点 IO。

    **`__write_node_page` 函数很可能是 F2FS 中实际提交节点页面 *写* IO 请求的底层函数。**  它可能封装了 BIO 结构体的创建、填充、提交等操作，并处理了同步/异步写回、错误处理、统计信息更新等细节。  **在 `f2fs_move_node_page` 函数中调用 `__write_node_page`，就实现了将节点页面 *迁移到新位置* 的写 IO 操作。**

*   **`__get_node_page(struct f2fs_sb_info *sbi, pgoff_t nid, struct page *parent, int start)`**
	* [_get_node_page](https://github.com/sigmanature/learn_os_note/edit/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/data.c/__get_node_page.md)

*   **`f2fs_get_node_page(struct f2fs_sb_info *sbi, pgoff_t nid)`**

```c
struct page *f2fs_get_node_page(struct f2fs_sb_info *sbi, pgoff_t nid)
{
	return __get_node_page(sbi, nid, NULL, 0);
}
```

*   **功能:** `f2fs_get_node_page` 函数是 `__get_node_page` 的 **简单封装**，用于 **获取 NID 对应的节点页面**。  它直接调用 `__get_node_page` 函数，并将 `parent` 和 `start` 参数设置为 `NULL` 和 `0` (表示没有父页面和起始偏移量，不进行预读优化)。

*   **`read_node_page(struct page *page, blk_opf_t op_flags)` (再次分析，Phase 2 调用)**

我们之前也分析过 `read_node_page` 函数，这里再次回顾，并注意在 Phase 2 中，`f2fs_get_node_page` 调用 `read_node_page` 时，`op_flags` 参数被设置为 `0` (非预读)。

```c
static int read_node_page(struct page *page, blk_opf_t op_flags)
{
	struct folio *folio = page_folio(page);
	struct f2fs_sb_info *sbi = F2FS_P_SB(page);
	struct node_info ni;
	struct f2fs_io_info fio = { ... };
	int err;

	if (folio_test_uptodate(folio)) {
		// ... (uptodate 状态检查和校验和验证) ...
	}

	err = f2fs_get_node_info(sbi, folio->index, &ni, false); // 获取节点信息
	if (err)
		return err;

	// ... (无效块地址检查) ...

	fio.new_blkaddr = fio.old_blkaddr = ni.blk_addr;
	err = f2fs_submit_page_bio(&fio); // 提交 BIO 请求

	// ... (IO 统计更新) ...

	return err;
}
```

*   **Phase 2 调用 `read_node_page` 的特点:**  在 `gc_node_segment` Phase 2 中，`f2fs_get_node_page` 函数调用 `read_node_page` 时，`op_flags` 参数被设置为 `0`。  **这意味着，Phase 2 的 `read_node_page` 调用是 *非预读* 的，只读取 *当前请求的页面*，不进行额外的预读操作。**  这与 Phase 1 的 `f2fs_ra_node_page` 调用 `read_node_page` 时，`op_flags` 参数设置为 `REQ_RAHEAD` (预读) 不同。  **Phase 1 预读 NAT 和节点页面，Phase 2 实际读取节点页面进行迁移。**

**2. `down_read`/`up_read` 并发 API**

你提到的 `down_read` 和 `up_read` 函数是 Linux 内核中 **读写信号量 (rw_semaphore)** 的 API，用于实现 **读写锁** 的功能。  **它们是通用的并发 API，广泛应用于内核代码中，用于保护共享资源的并发访问。**

*   **读写信号量 (rw_semaphore):**  读写信号量是一种 **比互斥锁 (mutex)** 更细粒度的并发控制机制。  它允许多个 **读者** 同时访问共享资源，但只允许一个 **写者** 独占访问共享资源。  读写信号量适用于 **读多写少** 的场景，可以提高并发性能。

*   **`down_read(struct rw_semaphore *sem)`:**  **获取读信号量**。  `down_read` 函数会 **阻塞** 当前线程，直到可以获取到读锁为止。  如果当前没有写者持有写锁，或者没有写者正在等待获取写锁，则 `down_read` 可以成功获取读锁并返回。  **允许多个线程同时成功 `down_read`，共享读锁。**

*   **`up_read(struct rw_semaphore *sem)`:**  **释放读信号量**。  `up_read` 函数会 **释放之前通过 `down_read` 获取的读锁**。  释放读锁后，其他等待读锁或写锁的线程可能会被唤醒。  **每次 `down_read` 必须对应一次 `up_read`，以保证信号量的正确使用。**

*   **`down_write(struct rw_semaphore *sem)` 和 `up_write(struct rw_semaphore *sem)`:**  与 `down_read`/`up_read` 类似，`down_write` 和 `up_write` 函数用于 **获取和释放 *写信号量***。  `down_write` 获取写锁，会 **独占** 信号量，**互斥所有其他读者和写者**。  `up_write` 释放写锁。  **写锁是排他锁，只允许一个线程持有。**

*   **`read_trylock(struct rw_semaphore *sem)` 和 `write_trylock(struct rw_semaphore *sem)`:**  `read_trylock` 和 `write_trylock` 函数是 **非阻塞的读写信号量获取函数**。  它们尝试获取读锁或写锁，如果 **立即可以获取到锁，则成功获取并返回非零值**；如果 **无法立即获取到锁 (例如，锁已被其他线程持有)，则 *不会阻塞*，而是立即返回 0 (失败)**。  `read_trylock` 和 `write_trylock` 适用于 **非阻塞的并发控制场景**。

*   **应用场景:**  在 F2FS 代码中，`down_read`/`up_read` 和 `down_write`/`up_write` 广泛应用于 **保护各种共享数据结构和资源**，例如：
    *   **SIT 缓存 (`sit_i->sentries`):**  使用 `sentry_lock` 读写信号量保护，允许多个线程同时读取 SIT 信息，但互斥写操作 (例如，SIT 更新)。
    *   **NAT 缓存 (R-tree 或类似结构):**  使用 `nat_tree_lock` 读写信号量保护，允许多个线程同时查找 NAT 信息，但互斥 NAT 缓存更新操作。
    *   **Journal 日志:**  使用 `journal_rwsem` 读写信号量保护，允许多个线程同时读取 Journal 日志，但互斥 Journal 日志写入操作。
    *   **Checkpoint 锁 (`sbi->cp_global_sem`):**  使用读写信号量保护 Checkpoint 相关的全局数据结构和操作。

**3. `old_blkaddr`/`new_blkaddr` 在 Phase 2 的作用**

你提出的关于 `old_blkaddr` 和 `new_blkaddr` 在 Phase 2 的作用的问题非常重要。  **在 `gc_node_segment` Phase 2 的代码中，以及 `f2fs_move_node_page` 函数中，并没有 *直接* 使用 `f2fs_io_info` 结构体的 `old_blkaddr` 和 `new_blkaddr` 字段。**  **`f2fs_io_info` 结构体主要在 `f2fs_submit_page_bio` 函数中被使用，用于描述 IO 操作的各种属性，包括目标块地址。**

*   **`f2fs_move_node_page` 的数据搬移逻辑:**  `f2fs_move_node_page` 函数的核心作用是 **将节点页面从 *旧位置* 迁移到 *新位置***。  虽然代码中没有显式地使用 `old_blkaddr` 和 `new_blkaddr`，但 **数据搬移的 *新旧位置信息* 是 *隐含* 在函数调用链中的**。

*   **隐含的新旧位置信息:**
    1.  **旧位置:**  `read_node_page` 函数在读取节点页面时，会调用 `f2fs_get_node_info` 函数 **获取节点页面的 *当前* 块地址**。  这个块地址就是 **节点页面的 *旧位置*** (在迁移之前的位置)。  `f2fs_get_node_info` 函数会将旧位置块地址存储在 `struct node_info` 结构体的 `blk_addr` 字段中，然后 `read_node_page` 函数会将 `ni.blk_addr` 赋值给 `f2fs_io_info` 结构体的 `fio.old_blkaddr` 和 `fio.new_blkaddr` 字段。  **因此，`fio.old_blkaddr` 和 `fio.new_blkaddr` *间接* 地记录了节点页面的旧位置。**
    2.  **新位置:**  在 `f2fs_move_node_page` 函数中，虽然没有显式指定 *新位置*，但 **新位置是在 `__write_node_page` 函数内部 *隐式确定* 的**。  `__write_node_page` 函数负责分配新的块地址，并将节点页面数据写入到新分配的块地址。  **新分配的块地址就是节点页面的 *新位置***。  `__write_node_page` 函数会将新位置块地址更新到 NAT 表中，建立 NID 到新位置的映射关系。

*   **`f2fs_io_info` 的作用:**  `f2fs_io_info` 结构体在数据搬移过程中，主要用于 **描述 *写 IO 请求***。  `f2fs_io_info` 的 `fio.new_blkaddr` 字段记录了 **写 IO 的目标块地址 (即新位置)**，`fio.page` 字段指向 **要写入的节点页面数据**。  `f2fs_submit_page_bio` 函数根据 `f2fs_io_info` 中的信息，构建并提交写 BIO 请求，将节点页面数据写入到新位置。

*   **总结:**  虽然 `gc_node_segment` Phase 2 和 `f2fs_move_node_page` 代码中没有直接操作 `old_blkaddr` 和 `new_blkaddr` 字段，但 **数据搬移的新旧位置信息是 *隐含* 在函数调用链和数据结构中的**。  `f2fs_io_info` 结构体作为 IO 请求描述符，**通过 `fio.new_blkaddr` 字段 *间接* 地体现了数据搬移的 *新位置* 信息**，并用于提交写 IO 请求，将节点页面数据写入到新位置，完成数据搬移操作。

**总结 Phase 2 分析:**  `gc_node_segment` Phase 2 阶段通过循环调用 `f2fs_get_node_page` 获取节点页面，然后调用 `f2fs_move_node_page` 函数 **将有效的节点页面迁移到新的位置**。  Phase 2 使用了 **读写信号量 (`sentry_lock`, `nat_tree_lock`, `journal_rwsem`)** 来保护共享资源的并发访问。  数据搬移的新旧位置信息 **隐含在函数调用链和数据结构中**，`f2fs_io_info` 结构体通过 `fio.new_blkaddr` 字段 **间接体现了数据搬移的新位置信息**，并用于提交写 IO 请求，完成数据搬移操作。

接下来，我们可以继续分析 `gc_data_segment` 函数，并开始思考如何针对 Phase 2 进行 Folio 优化，以及如何更有效地利用 Folio 的批量特性来加速节点和数据页面的垃圾回收过程。



