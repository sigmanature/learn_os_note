**1. `gc_node_segment(struct f2fs_sb_info *sbi, struct f2fs_summary *sum, unsigned int segno, int gc_type)`**

```c
static int gc_node_segment(struct f2fs_sb_info *sbi,
		struct f2fs_summary *sum, unsigned int segno, int gc_type)
{
	struct f2fs_summary *entry;
	block_t start_addr;
	int off;
	int phase = 0;
	bool fggc = (gc_type == FG_GC);
	int submitted = 0;
	unsigned int usable_blks_in_seg = f2fs_usable_blks_in_seg(sbi, segno);

	start_addr = START_BLOCK(sbi, segno);

next_step:
	entry = sum;

	if (fggc && phase == 2)
		atomic_inc(&sbi->wb_sync_req[NODE]);

	for (off = 0; off < usable_blks_in_seg; off++, entry++) {
		nid_t nid = le32_to_cpu(entry->nid);
		struct page *node_page;
		struct node_info ni;
		int err;

		/* stop BG_GC if there is not enough free sections. */
		if (gc_type == BG_GC && has_not_enough_free_secs(sbi, 0, 0))
			return submitted;

		if (check_valid_map(sbi, segno, off) == 0)
			continue;

		if (phase == 0) {
			f2fs_ra_meta_pages(sbi, NAT_BLOCK_OFFSET(nid), 1,
							META_NAT, true);
			continue;
		}

		if (phase == 1) {
			f2fs_ra_node_page(sbi, nid);
			continue;
		}

		/* phase == 2 */
		node_page = f2fs_get_node_page(sbi, nid);
		if (IS_ERR(node_page))
			continue;

		/* block may become invalid during f2fs_get_node_page */
		if (check_valid_map(sbi, segno, off) == 0) {
			f2fs_put_page(node_page, 1);
			continue;
		}

		if (f2fs_get_node_info(sbi, nid, &ni, false)) {
			f2fs_put_page(node_page, 1);
			continue;
		}

		if (ni.blk_addr != start_addr + off) {
			f2fs_put_page(node_page, 1);
			continue;
		}

		err = f2fs_move_node_page(node_page, gc_type);
		if (!err && gc_type == FG_GC)
			submitted++;
		stat_inc_node_blk_count(sbi, 1, gc_type);
	}

	if (++phase < 3)
		goto next_step;

	if (fggc)
		atomic_dec(&sbi->wb_sync_req[NODE]);
	return submitted;
}
```

*   **功能概述:** `gc_node_segment` 函数负责处理 **节点段** 的垃圾回收。它遍历给定 segment (`segno`) 的 Summary Block (`sum`) 中的条目，检查每个节点块，并将有效的节点块迁移到新的位置。

*   **参数:**
    *   `struct f2fs_sb_info *sbi`: 文件系统超级块信息。
    *   `struct f2fs_summary *sum`: 指向当前 segment 的 Summary Block 的指针。
    *   `unsigned int segno`: 当前正在处理的 segment 号。
    *   `int gc_type`: 垃圾回收类型 (`FG_GC` 或 `BG_GC`)。

*   **变量初始化:**
    *   `struct f2fs_summary *entry;`: 用于遍历 Summary Block 条目的指针。
    *   `block_t start_addr;`: 当前 segment 的起始块地址，通过 `START_BLOCK` 宏计算。
    *   `int off;`: 块在 segment 内的偏移量。
    *   `int phase = 0;`:  垃圾回收的阶段，`gc_node_segment` 分为三个阶段执行。
    *   `bool fggc = (gc_type == FG_GC);`:  标记是否为前台 GC。
    *   `int submitted = 0;`:  记录提交的 IO 请求数量。
    *   `unsigned int usable_blks_in_seg = f2fs_usable_blks_in_seg(sbi, segno);`: 获取当前 segment 中可用的块数量，考虑了 zoned device 的情况。

*   **计算起始块地址 `start_addr = START_BLOCK(sbi, segno);`**:  通过 `START_BLOCK` 宏计算当前 segment 的起始块地址。我们稍后会详细分析 `START_BLOCK` 宏。因为涉及到了使用将逻辑segment号转化为物理的segment号这一行为,这意味着我们在do_garbage_collect函数中计算和使用的都是逻辑的segment
号码。

*   **三阶段循环 (`next_step` 标签和 `phase` 变量):**  `gc_node_segment` 函数使用 `phase` 变量和 `goto next_step` 实现了三阶段的处理流程。
    *   **`next_step:` 标签:**  循环的入口点。
    *   **`entry = sum;`:**  每次进入 `next_step`，`entry` 指针都重置为 Summary Block 的起始位置 `sum`，以便从头开始遍历 Summary Block 的条目。
    *   **`if (fggc && phase == 2) atomic_inc(&sbi->wb_sync_req[NODE]);`:**  如果为前台 GC 且当前为第三阶段 (`phase == 2`)，则增加节点类型的写回同步请求计数器 `sbi->wb_sync_req[NODE]`。这可能用于前台 GC 的同步控制。
    *   **`for (off = 0; off < usable_blks_in_seg; off++, entry++) { ... }` 循环:**  遍历当前 segment 的 Summary Block 条目。
        *   `off`:  块在 segment 内的偏移量，从 0 递增到 `usable_blks_in_seg - 1`。
        *   `entry++`:  `entry` 指针递增，指向下一个 Summary Block 条目。
        *   **`nid_t nid = le32_to_cpu(entry->nid);`:**  从当前 Summary Block 条目 `entry` 中读取节点 ID (`nid`)，并进行字节序转换。
        *   **`struct page *node_page; struct node_info ni; int err;`:**  声明局部变量，用于存储节点页面、节点信息和错误码。
        *   **`if (gc_type == BG_GC && has_not_enough_free_secs(sbi, 0, 0)) return submitted;`:**  对于后台 GC，检查是否有足够的空闲 section。如果没有，则提前返回，停止后台 GC。
        *   **`if (check_valid_map(sbi, segno, off) == 0) continue;`:**  检查当前块是否有效。`check_valid_map` 函数检查块的有效性位图。如果块无效，则跳过当前块，处理下一个块。
        *   **`if (phase == 0) { ... }` (第一阶段):**
            ```c
            if (phase == 0) {
                f2fs_ra_meta_pages(sbi, NAT_BLOCK_OFFSET(nid), 1,
                                META_NAT, true);
                continue;
            }
            ```
            *   **元数据预读 (NAT):**  在第一阶段，对 **Node Address Table (NAT)** 进行预读。
            *   `f2fs_ra_meta_pages(sbi, NAT_BLOCK_OFFSET(nid), 1, META_NAT, true);`:  调用 `f2fs_ra_meta_pages` 函数预读 NAT 页面。
                *   `NAT_BLOCK_OFFSET(nid)`:  根据节点 ID (`nid`) 计算 NAT 块的偏移地址。
                *   `1`:  预读的页面数量为 1。
                *   `META_NAT`:  预读的元数据类型为 NAT。
                *   `true`:  `sync` 参数为 true，表示同步预读 (虽然这里是预读，但 `sync=true` 可能会影响预读的优先级或行为)。
            *   `continue;`:  完成预读后，跳过当前块的后续处理，继续处理下一个块。
        *   **`if (phase == 1) { ... }` (第二阶段):**
            ```c
            if (phase == 1) {
                f2fs_ra_node_page(sbi, nid);
                continue;
            }
            ```
            *   **节点页面预读:**  在第二阶段，对 **节点页面** 本身进行预读。
            *   `f2fs_ra_node_page(sbi, nid);`:  调用 `f2fs_ra_node_page` 函数预读节点页面。
            *   `continue;`:  完成预读后，跳过当前块的后续处理，继续处理下一个块。
        *   **`if (phase == 2) { ... }` (第三阶段):**
            ```c
            /* phase == 2 */
            node_page = f2fs_get_node_page(sbi, nid);
            if (IS_ERR(node_page))
                continue;

            /* block may become invalid during f2fs_get_node_page */
            if (check_valid_map(sbi, segno, off) == 0) {
                f2fs_put_page(node_page, 1);
                continue;
            }

            if (f2fs_get_node_info(sbi, nid, &ni, false)) {
                f2fs_put_page(node_page, 1);
                continue;
            }

            if (ni.blk_addr != start_addr + off) {
                f2fs_put_page(node_page, 1);
                continue;
            }

            err = f2fs_move_node_page(node_page, gc_type);
            if (!err && gc_type == FG_GC)
                submitted++;
            stat_inc_node_blk_count(sbi, 1, gc_type);
            ```
            *   **获取节点页面:**  `node_page = f2fs_get_node_page(sbi, nid);`:  调用 `f2fs_get_node_page` 函数获取节点页面。
            *   **错误处理:**  `if (IS_ERR(node_page)) continue;`:  如果 `f2fs_get_node_page` 返回错误，则跳过当前块，处理下一个块。
            *   **再次检查块有效性:**  `if (check_valid_map(sbi, segno, off) == 0) { ... }`:  在 `f2fs_get_node_page` 调用之后，再次检查块的有效性。因为在 `f2fs_get_node_page` 执行期间，块可能变得无效 (例如，被其他操作回收)。如果块无效，则释放 `node_page` 并跳过。**注意!第三阶段进行的是有效数据的搬移!!gc实际上是将有效数据合并在一起使得无效数据,或者可被覆写的数据空间变得更大更连续。而不是直接"回收"无效数据**
            *   **获取节点信息:**  `if (f2fs_get_node_info(sbi, nid, &ni, false)) { ... }`:  调用 `f2fs_get_node_info` 函数获取节点信息 (`struct node_info`)。如果获取失败，则释放 `node_page` 并跳过。
            *   **地址一致性检查:**  `if (ni.blk_addr != start_addr + off) { ... }`:  检查从 `f2fs_get_node_info` 获取的节点块地址 `ni.blk_addr` 是否与当前 segment 的预期块地址 `start_addr + off` 一致。如果不一致，则释放 `node_page` 并跳过。 **这个检查是为了确保 Summary Block 中记录的节点地址与实际的节点信息一致性。**
            *   **迁移节点页面:**  `err = f2fs_move_node_page(node_page, gc_type);`:  调用 `f2fs_move_node_page` 函数 **迁移节点页面**。这是实际执行数据迁移的关键步骤。
            *   **统计信息更新:**
                *   `if (!err && gc_type == FG_GC) submitted++;`:  如果 `f2fs_move_node_page` 成功 (没有错误) 且为前台 GC，则增加 `submitted` 计数器。
                *   `stat_inc_node_blk_count(sbi, 1, gc_type);`:  增加节点块计数统计。

    *   **`if (++phase < 3) goto next_step;`:**  在遍历完当前 segment 的所有块后，`phase` 递增。如果 `phase` 小于 3，则跳转回 `next_step` 标签，开始下一个阶段的处理。  **`gc_node_segment` 函数总共执行三个阶段 (phase 0, 1, 2)。**

*   **前台 GC 同步计数器递减:**  `if (fggc) atomic_dec(&sbi->wb_sync_req[NODE]);`:  如果为前台 GC，在完成所有三个阶段后，递减节点类型的写回同步请求计数器 `sbi->wb_sync_req[NODE]`。

*   **返回值:**  `return submitted;`:  返回 `submitted` 计数器，表示在前台 GC 中成功迁移的节点块数量。

*   **总结 `gc_node_segment`:**  `gc_node_segment` 函数通过 **三阶段处理** 和 **遍历 Summary Block 条目** 的方式，实现了节点段的垃圾回收。  **三个阶段分别负责 NAT 预读、节点页面预读和实际的节点页面迁移。**  函数内部进行了多重检查，包括块有效性检查、节点信息获取、地址一致性检查，确保数据迁移的正确性和可靠性。  **实际的 IO 提交操作主要发生在 `f2fs_ra_meta_pages`, `f2fs_ra_node_page`, `f2fs_get_node_page`, 和 `f2fs_move_node_page` 等被调用的函数中。**

**2. `f2fs_usable_blks_in_seg(struct f2fs_sb_info *sbi, unsigned int segno)`**

```c
unsigned int f2fs_usable_blks_in_seg(struct f2fs_sb_info *sbi,
					unsigned int segno)
{
	if (f2fs_sb_has_blkzoned(sbi))
		return f2fs_usable_zone_blks_in_seg(sbi, segno);

	return BLKS_PER_SEG(sbi);
}
```

*   **功能概述:**  `f2fs_usable_blks_in_seg` 函数用于 **获取给定 segment (`segno`) 中可用的数据块数量**。

*   **Zoned Block Device 处理:**
    ```c
    if (f2fs_sb_has_blkzoned(sbi))
        return f2fs_usable_zone_blks_in_seg(sbi, segno);
    ```
    *   `f2fs_sb_has_blkzoned(sbi)`:  检查文件系统是否运行在 zoned block device 上。
    *   `f2fs_usable_zone_blks_in_seg(sbi, segno)`:  如果是 zoned device，则调用 `f2fs_usable_zone_blks_in_seg` 函数获取 zoned device 中 segment 的可用块数量。 zoned device 的可用块数量可能小于非 zoned device。

*   **非 Zoned Block Device 处理:**
    ```c
    return BLKS_PER_SEG(sbi);
    ```
    *   `BLKS_PER_SEG(sbi)`:  如果不是 zoned device，则直接返回 `BLKS_PER_SEG(sbi)` 宏定义的值，表示每个 segment 的块数量。

*   **总结 `f2fs_usable_blks_in_seg`:**  `f2fs_usable_blks_in_seg` 函数根据文件系统是否运行在 zoned block device 上，返回不同的可用块数量。 对于 zoned device，需要考虑 zone 的限制，可能每个 segment 的可用块数量会减少。

**3. `START_BLOCK(sbi, segno)` 宏**

```c
#define START_BLOCK(sbi, segno)	(SEG0_BLKADDR(sbi) +			\
	 (SEGS_TO_BLKS(sbi, GET_R2L_SEGNO(FREE_I(sbi), segno))))
#define SEG0_BLKADDR(sbi)						\
	(SM_I(sbi) ? SM_I(sbi)->seg0_blkaddr : 				\
		le32_to_cpu(F2FS_RAW_SUPER(sbi)->segment0_blkaddr))
/* Definitions to access f2fs_sb_info */
#define SEGS_TO_BLKS(sbi, segs)					\
		((segs) << (sbi)->log_blocks_per_seg)
#define GET_R2L_SEGNO(free_i, segno)	((segno) + (free_i)->start_segno)
static inline struct free_segmap_info *FREE_I(struct f2fs_sb_info *sbi)
{
	return (struct free_segmap_info *)(SM_I(sbi)->free_info);
}
```

*   **功能概述:** `START_BLOCK(sbi, segno)` 宏用于 **计算给定 segment 号 (`segno`) 的起始块地址**。  它涉及到 F2FS 的 R2L (Reverse Translation Layer) 映射和 segment 组织方式。

*   **`SEG0_BLKADDR(sbi)` 宏:**
    ```c
    #define SEG0_BLKADDR(sbi)						\
    	(SM_I(sbi) ? SM_I(sbi)->seg0_blkaddr : 				\
    		le32_to_cpu(F2FS_RAW_SUPER(sbi)->segment0_blkaddr))
    ```
    *   `SM_I(sbi)`:  获取 `segment_manager_info` 结构体指针。
    *   `SM_I(sbi) ? SM_I(sbi)->seg0_blkaddr : ...`:  如果 `segment_manager_info` 存在 (`SM_I(sbi)` 为真)，则使用 `SM_I(sbi)->seg0_blkaddr`，否则使用 `F2FS_RAW_SUPER(sbi)->segment0_blkaddr`。  这表明 `seg0_blkaddr` 可能从两个地方获取，优先从 `segment_manager_info` 获取。 `seg0_blkaddr` 应该是 **segment 0 的起始块地址**。
    *   `le32_to_cpu(F2FS_RAW_SUPER(sbi)->segment0_blkaddr)`:  如果从 `F2FS_RAW_SUPER(sbi)` 获取，则需要进行字节序转换。

*   **`FREE_I(sbi)` 宏:**
    ```c
    static inline struct free_segmap_info *FREE_I(struct f2fs_sb_info *sbi)
    {
    	return (struct free_segmap_info *)(SM_I(sbi)->free_info);
    }
    ```
    *   `FREE_I(sbi)`:  获取 `free_segmap_info` 结构体指针，用于管理空闲 segment 信息。  它通过 `SM_I(sbi)->free_info` 获取。

*   **`GET_R2L_SEGNO(FREE_I(sbi), segno)` 宏:**
    ```c
    #define GET_R2L_SEGNO(free_i, segno)	((segno) + (free_i)->start_segno)
    ```
    *   `GET_R2L_SEGNO(free_i, segno)`:  计算 **Reverse Translation Layer (R2L) 映射后的 segment 号**。
    *   `(segno) + (free_i)->start_segno`:  将给定的 `segno` 加上 `free_segmap_info` 中的 `start_segno`。  `start_segno` 可能表示 R2L 映射的起始 segment 号。  **R2L 映射用于将逻辑 segment 号映射到物理 segment 号，以实现磨损均衡和灵活的 segment 管理。**

*   **`SEGS_TO_BLKS(sbi, segs)` 宏:**
    ```c
    #define SEGS_TO_BLKS(sbi, segs)					\
    		((segs) << (sbi)->log_blocks_per_seg)
    ```
    *   `SEGS_TO_BLKS(sbi, segs)`:  将 segment 数量 (`segs`) 转换为块数量。
    *   `(segs) << (sbi)->log_blocks_per_seg`:  将 `segs` 左移 `(sbi)->log_blocks_per_seg` 位。 `(sbi)->log_blocks_per_seg` 应该表示每个 segment 的块数量的以 2 为底的对数。  **左移操作等价于乘以 2 的 `log_blocks_per_seg` 次方，即乘以每个 segment 的块数量。**

*   **`START_BLOCK(sbi, segno)` 宏的完整计算:**
    ```c
    #define START_BLOCK(sbi, segno)	(SEG0_BLKADDR(sbi) +			\
    	 (SEGS_TO_BLKS(sbi, GET_R2L_SEGNO(FREE_I(sbi), segno))))
    ```
    *   `SEG0_BLKADDR(sbi)`:  获取 segment 0 的起始块地址。
    *   `GET_R2L_SEGNO(FREE_I(sbi), segno)`:  计算 R2L 映射后的 segment 号。
    *   `SEGS_TO_BLKS(sbi, ...)`:  将 R2L 映射后的 segment 号转换为块数量，得到从 segment 0 起始地址到目标 segment 起始地址的块偏移量。
    *   `SEG0_BLKADDR(sbi) + ...`:  将 segment 0 的起始块地址加上块偏移量，得到目标 segment (`segno`) 的起始块地址。

**4. `f2fs_ra_meta_pages(struct f2fs_sb_info *sbi, block_t start, int nrpages, int type, bool sync)`**
* 调用处
```C
if (phase == 0) {
			f2fs_ra_meta_pages(sbi, NAT_BLOCK_OFFSET(nid), 1,
							META_NAT, true);
			continue;
		}
```
[f2fs_ra_meta_pages](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/checkpoint.c/f2fs_ra_meta_pages.md)<br>
**5. `f2fs_ra_node_page(struct f2fs_sb_info *sbi, nid_t nid)`**
[f2fs_ra_node_page](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/node.c/f2fs_ra_node_page.md)<br>
**6. `read_node_page(struct page *page, blk_opf_t op_flags)`**
[read_node_page](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/node.c/read_node_page.md)<br>
**7. `f2fs_move_node_page(struct page *node_page, int gc_type)`**
[f2fs_move_node_page](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/node.c/f2fs_move_node_page.md)<br>
