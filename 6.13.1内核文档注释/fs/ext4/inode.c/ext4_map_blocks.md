 **相关函数**
 * [ext4_mpage_readpages](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/ext4/inode.c/ext4_mpage_readpages.md)
 好的，我们来深入分析 `ext4_map_blocks` 函数，并着重理解它如何处理 extent 和传统的间接块映射，以及它在 `ext4_mpage_readpages` 中扮演的角色。

```c
int ext4_map_blocks(handle_t *handle, struct inode *inode,
		    struct ext4_map_blocks *map, int flags)
{
	struct extent_status es;
	int retval;
	int ret = 0;
#ifdef ES_AGGRESSIVE_TEST
	struct ext4_map_blocks orig_map;

	memcpy(&orig_map, map, sizeof(*map));
#endif

	map->m_flags = 0;
	ext_debug(inode, "flag 0x%x, max_blocks %u, logical block %lu\n",
		  flags, map->m_len, (unsigned long) map->m_lblk);
```
**函数开始和初始化:**
- `int ext4_map_blocks(...)`: 函数定义，返回类型为 `int`，表示操作结果。
    - `handle_t *handle`:  事务句柄，用于在事务上下文中进行操作。如果为 `NULL`，则不在事务上下文中。
    - `struct inode *inode`:  目标文件的 inode 结构体指针。
    - `struct ext4_map_blocks *map`:  指向 `ext4_map_blocks` 结构体的指针，用于输入请求参数（如逻辑块号 `m_lblk` 和长度 `m_len`），并输出映射结果（如物理块号 `m_pblk` 和标志 `m_flags`）。
    - `int flags`:  标志位，控制函数行为，例如是否创建块、是否只查询缓存等。
- `struct extent_status es;`: 声明 `extent_status` 结构体变量 `es`，用于存储 extent 状态查找的结果。
- `int retval; int ret = 0;`: 声明整型变量 `retval` 用于存储函数返回值，`ret` 用于中间错误值传递。
- `#ifdef ES_AGGRESSIVE_TEST ... #endif`:  条件编译代码块，用于 aggressive extent status 测试，通常在正常运行时不编译。
- `map->m_flags = 0;`:  初始化 `map->m_flags` 为 0，清除之前的标志。
- `ext_debug(...)`:  调试信息输出，打印传入的标志、请求长度和逻辑块号。

```c
	/*
	 * ext4_map_blocks returns an int, and m_len is an unsigned int
	 */
	if (unlikely(map->m_len > INT_MAX))
		map->m_len = INT_MAX;

	/* We can handle the block number less than EXT_MAX_BLOCKS */
	if (unlikely(map->m_lblk >= EXT_MAX_BLOCKS))
		return -EFSCORRUPTED;
```
**输入参数检查:**
- `if (unlikely(map->m_len > INT_MAX)) map->m_len = INT_MAX;`:  由于 `ext4_map_blocks` 返回 `int` 类型的长度，这里限制请求的长度 `map->m_len` 不超过 `INT_MAX`，以避免溢出。 `unlikely()` 是 likely/unlikely 分支预测宏，提示编译器这种情况不太可能发生。
- `if (unlikely(map->m_lblk >= EXT_MAX_BLOCKS)) return -EFSCORRUPTED;`: 检查请求的逻辑块号 `map->m_lblk` 是否超过了 `EXT_MAX_BLOCKS` 限制。如果超过，返回 `-EFSCORRUPTED` 错误，表示文件系统损坏。 `EXT_MAX_BLOCKS` 定义了 ext4 文件系统可以处理的最大块号。

```c
	/* Lookup extent status tree firstly */
	if (!(EXT4_SB(inode->i_sb)->s_mount_state & EXT4_FC_REPLAY) &&
	    ext4_es_lookup_extent(inode, map->m_lblk, NULL, &es)) {
		if (ext4_es_is_written(&es) || ext4_es_is_unwritten(&es)) {
			map->m_pblk = ext4_es_pblock(&es) +
					map->m_lblk - es.es_lblk;
			map->m_flags |= ext4_es_is_written(&es) ?
					EXT4_MAP_MAPPED : EXT4_MAP_UNWRITTEN;
			retval = es.es_len - (map->m_lblk - es.es_lblk);
			if (retval > map->m_len)
				retval = map->m_len;
			map->m_len = retval;
		} else if (ext4_es_is_delayed(&es) || ext4_es_is_hole(&es)) {
			map->m_pblk = 0;
			map->m_flags |= ext4_es_is_delayed(&es) ?
					EXT4_MAP_DELAYED : 0;
			retval = es.es_len - (map->m_lblk - es.es_lblk);
			if (retval > map->m_len)
				retval = map->m_len;
			map->m_len = retval;
			retval = 0;
		} else {
			BUG();
		}

		if (flags & EXT4_GET_BLOCKS_CACHED_NOWAIT)
			return retval;
#ifdef ES_AGGRESSIVE_TEST
		ext4_map_blocks_es_recheck(handle, inode, map,
					   &orig_map, flags);
#endif
		goto found;
	}
```
**Extent Status Tree (ES Tree) 查找:**
- `if (!(EXT4_SB(inode->i_sb)->s_mount_state & EXT4_FC_REPLAY) && ext4_es_lookup_extent(inode, map->m_lblk, NULL, &es))`:  首先尝试在 extent status tree 中查找 extent 信息。
    - `!(EXT4_SB(inode->i_sb)->s_mount_state & EXT4_FC_REPLAY)`:  检查文件系统是否处于快速清理重放 (fast-commit replay) 状态。如果是，则跳过 ES tree 查找。
    - `ext4_es_lookup_extent(inode, map->m_lblk, NULL, &es)`:  调用 `ext4_es_lookup_extent` 函数在 inode 的 extent status tree 中查找逻辑块号 `map->m_lblk` 的 extent 状态。
        - `inode`:  目标 inode。
        - `map->m_lblk`:  要查找的逻辑块号。
        - `NULL`:  父 extent status 结构体指针，这里为 `NULL` 表示查找顶层 extent。
        - `&es`:  指向 `extent_status` 结构体 `es` 的指针，用于接收查找结果。
    - 如果 `ext4_es_lookup_extent` 返回非零值（表示找到 extent），则进入 `if` 代码块。
- **处理 ES Tree 查找结果:**
    - `if (ext4_es_is_written(&es) || ext4_es_is_unwritten(&es))`:  如果找到的 extent 是已写入 (`written`) 或未写入 (`unwritten`) 状态。
        - `map->m_pblk = ext4_es_pblock(&es) + (map->m_lblk - es.es_lblk);`:  计算物理块号 `map->m_pblk`。`ext4_es_pblock(&es)` 返回 extent 的起始物理块号，加上逻辑块号 `map->m_lblk` 相对于 extent 起始逻辑块号 `es.es_lblk` 的偏移量，得到目标逻辑块的物理块号。
        - `map->m_flags |= ext4_es_is_written(&es) ? EXT4_MAP_MAPPED : EXT4_MAP_UNWRITTEN;`:  设置 `map->m_flags`。如果是已写入 extent，设置 `EXT4_MAP_MAPPED` 标志；如果是未写入 extent，设置 `EXT4_MAP_UNWRITTEN` 标志。
        - `retval = es.es_len - (map->m_lblk - es.es_lblk);`:  计算返回的块数 `retval`。`es.es_len` 是 extent 的长度，减去逻辑块号偏移量，得到从 `map->m_lblk` 开始到 extent 结束的剩余长度。
        - `if (retval > map->m_len) retval = map->m_len;`:  限制返回的块数 `retval` 不超过请求的长度 `map->m_len`。
        - `map->m_len = retval;`:  更新 `map->m_len` 为实际返回的块数。
    - `else if (ext4_es_is_delayed(&es) || ext4_es_is_hole(&es))`:  如果找到的 extent 是延迟分配 (`delayed`) 或空洞 (`hole`) 状态。
        - `map->m_pblk = 0;`:  物理块号设置为 0，因为延迟分配和空洞没有实际的物理块。
        - `map->m_flags |= ext4_es_is_delayed(&es) ? EXT4_MAP_DELAYED : 0;`:  设置 `map->m_flags`。如果是延迟分配 extent，设置 `EXT4_MAP_DELAYED` 标志；如果是空洞 extent，不设置标志（`m_flags` 保持为 0，表示未映射）。
        - `retval = es.es_len - (map->m_lblk - es.es_lblk);`:  计算返回的块数 `retval`，与上面类似。
        - `if (retval > map->m_len) retval = map->m_len;`:  限制返回的块数。
        - `map->m_len = retval;`:  更新 `map->m_len`。
        - `retval = 0;`:  对于延迟分配和空洞，`ext4_map_blocks` 的返回值设置为 0，表示没有实际映射到物理块。
    - `else { BUG(); }`:  如果 extent 状态不是 written, unwritten, delayed, hole 中的任何一种，则触发 BUG，表示代码逻辑错误。
- `if (flags & EXT4_GET_BLOCKS_CACHED_NOWAIT) return retval;`:  如果设置了 `EXT4_GET_BLOCKS_CACHED_NOWAIT` 标志，表示只查询缓存，不进行阻塞操作。在这种情况下，直接返回 `retval`，即 ES tree 查找的结果。
- `#ifdef ES_AGGRESSIVE_TEST ... #endif`:  条件编译代码块，用于 aggressive extent status 测试重检查。
- `goto found;`:  跳转到 `found` 标签，表示已经找到映射信息（从 ES tree 或后续的查询操作）。

**总结 ES Tree 查找部分:**

这部分代码首先尝试从 extent status tree 中快速查找块映射信息。ES tree 是一个缓存，用于加速 extent 查找。如果找到 extent 信息，并且是 written 或 unwritten 状态，则直接返回物理块号和 `EXT4_MAP_MAPPED` 或 `EXT4_MAP_UNWRITTEN` 标志。如果是 delayed 或 hole 状态，则返回 `EXT4_MAP_DELAYED` 标志或不设置标志，物理块号为 0，返回值 `retval` 为 0。 如果设置了 `EXT4_GET_BLOCKS_CACHED_NOWAIT` 标志，则只进行 ES tree 查找，找不到就直接返回 0。

```c
	/*
	 * In the query cache no-wait mode, nothing we can do more if we
	 * cannot find extent in the cache.
	 */
	if (flags & EXT4_GET_BLOCKS_CACHED_NOWAIT)
		return 0;

	/*
	 * Try to see if we can get the block without requesting a new
	 * file system block.
	 */
	down_read(&EXT4_I(inode)->i_data_sem);
	retval = ext4_map_query_blocks(handle, inode, map);
	up_read((&EXT4_I(inode)->i_data_sem));

found:
	if (retval > 0 && map->m_flags & EXT4_MAP_MAPPED) {
		ret = check_block_validity(inode, map);
		if (ret != 0)
			return ret;
	}
```
**非 ES Tree 缓存查询 (ext4_map_query_blocks):**
- `if (flags & EXT4_GET_BLOCKS_CACHED_NOWAIT) return 0;`:  再次检查 `EXT4_GET_BLOCKS_CACHED_NOWAIT` 标志。如果设置了，并且之前 ES tree 查找失败，则直接返回 0，表示在 no-wait 模式下无法找到映射。
- `down_read(&EXT4_I(inode)->i_data_sem);`:  获取 inode 的 `i_data_sem` 读锁。`i_data_sem` 保护 inode 的数据块映射信息，读锁允许多个读者并发访问，但排斥写者。
- `retval = ext4_map_query_blocks(handle, inode, map);`:  调用 `ext4_map_query_blocks` 函数查询块映射。这个函数会查找 inode 的 extent tree 或间接块映射结构，尝试找到已有的块映射，但不进行任何分配操作。
    - `handle`:  事务句柄。
    - `inode`:  目标 inode。
    - `map`:  `ext4_map_blocks` 结构体，包含请求参数和接收结果。
- `up_read((&EXT4_I(inode)->i_data_sem));`:  释放 `i_data_sem` 读锁。
- `found:` 标签:  代码执行到这里，表示已经尝试了 ES tree 查找和 `ext4_map_query_blocks` 查询。
- `if (retval > 0 && map->m_flags & EXT4_MAP_MAPPED)`:  如果 `retval` 大于 0（表示找到映射）并且设置了 `EXT4_MAP_MAPPED` 标志。
    - `ret = check_block_validity(inode, map);`:  调用 `check_block_validity` 函数检查找到的块映射是否有效。
    - `if (ret != 0) return ret;`:  如果块映射无效，返回错误码 `ret`。

**总结 `ext4_map_query_blocks` 部分:**

如果 ES tree 查找失败，或者没有设置 `EXT4_GET_BLOCKS_CACHED_NOWAIT` 标志，则会调用 `ext4_map_query_blocks` 进行更全面的块映射查询。这个函数会根据 inode 的类型（extent-based 或 indirect-based）选择不同的查找方法（extent tree 或间接块映射）。查询操作需要获取 `i_data_sem` 读锁来保护 inode 的数据结构。 找到映射后，会进行有效性检查。

```c
	/* If it is only a block(s) look up */
	if ((flags & EXT4_GET_BLOCKS_CREATE) == 0)
		return retval;

	/*
	 * Returns if the blocks have already allocated
	 *
	 * Note that if blocks have been preallocated
	 * ext4_ext_map_blocks() returns with buffer head unmapped
	 */
	if (retval > 0 && map->m_flags & EXT4_MAP_MAPPED)
		/*
		 * If we need to convert extent to unwritten
		 * we continue and do the actual work in
		 * ext4_ext_map_blocks()
		 */
		if (!(flags & EXT4_GET_BLOCKS_CONVERT_UNWRITTEN))
			return retval;
```
**处理只查询的情况和已分配块的情况:**
- `if ((flags & EXT4_GET_BLOCKS_CREATE) == 0) return retval;`:  如果标志 `flags` 中没有设置 `EXT4_GET_BLOCKS_CREATE`，表示只进行块查找，不进行块分配。在这种情况下，直接返回之前查询操作的结果 `retval`。
- `if (retval > 0 && map->m_flags & EXT4_MAP_MAPPED)`:  如果之前查询操作找到了映射的块（`retval > 0` 且设置了 `EXT4_MAP_MAPPED` 标志）。
    - `if (!(flags & EXT4_GET_BLOCKS_CONVERT_UNWRITTEN)) return retval;`:  如果标志 `flags` 中没有设置 `EXT4_GET_BLOCKS_CONVERT_UNWRITTEN`，表示不需要将 extent 转换为 unwritten 状态。在这种情况下，直接返回 `retval`，因为已经找到了映射的块，并且不需要进行分配或转换操作。

**总结只查询和已分配块处理部分:**

如果调用 `ext4_map_blocks` 的目的是只进行块查找（没有 `EXT4_GET_BLOCKS_CREATE` 标志），或者已经找到了映射的块，并且不需要进行 unwritten 转换，则函数会提前返回，不再进行后续的块分配操作。

```c
	/*
	 * New blocks allocate and/or writing to unwritten extent
	 * will possibly result in updating i_data, so we take
	 * the write lock of i_data_sem, and call get_block()
	 * with create == 1 flag.
	 */
	down_write(&EXT4_I(inode)->i_data_sem);
	retval = ext4_map_create_blocks(handle, inode, map, flags);
	up_write((&EXT4_I(inode)->i_data_sem));
	if (retval > 0 && map->m_flags & EXT4_MAP_MAPPED) {
		ret = check_block_validity(inode, map);
		if (ret != 0)
			return ret;
```
**块分配 (ext4_map_create_blocks):**
- `down_write(&EXT4_I(inode)->i_data_sem);`:  获取 inode 的 `i_data_sem` 写锁。由于块分配或 unwritten extent 写入操作可能会修改 inode 的数据块映射信息，需要排他性的写锁来保护数据结构。
- `retval = ext4_map_create_blocks(handle, inode, map, flags);`:  调用 `ext4_map_create_blocks` 函数进行块分配或 unwritten extent 写入。
    - `handle`:  事务句柄。
    - `inode`:  目标 inode。
    - `map`:  `ext4_map_blocks` 结构体，包含请求参数和接收结果。
    - `flags`:  标志位，控制分配行为，例如是否预分配、是否需要零填充等。
- `up_write((&EXT4_I(inode)->i_data_sem));`:  释放 `i_data_sem` 写锁。
- `if (retval > 0 && map->m_flags & EXT4_MAP_MAPPED)`:  如果 `ext4_map_create_blocks` 返回值大于 0（表示成功分配或写入）并且设置了 `EXT4_MAP_MAPPED` 标志。
    - `ret = check_block_validity(inode, map);`:  再次调用 `check_block_validity` 检查新分配或写入的块映射是否有效。
    - `if (ret != 0) return ret;`:  如果块映射无效，返回错误码 `ret`。

**总结 `ext4_map_create_blocks` 部分:**

如果需要进行块分配（设置了 `EXT4_GET_BLOCKS_CREATE` 标志），并且之前的查询操作没有找到合适的映射，或者需要将 extent 转换为 unwritten 状态，则会调用 `ext4_map_create_blocks` 函数。这个函数负责实际的块分配操作，并根据 inode 类型选择 extent-based 或 indirect-based 的分配方法。分配操作需要获取 `i_data_sem` 写锁。

```c
		/*
		 * Inodes with freshly allocated blocks where contents will be
		 * visible after transaction commit must be on transaction's
		 * ordered data list.
		 */
		if (map->m_flags & EXT4_MAP_NEW &&
		    !(map->m_flags & EXT4_MAP_UNWRITTEN) &&
		    !(flags & EXT4_GET_BLOCKS_ZERO) &&
		    !ext4_is_quota_file(inode) &&
		    ext4_should_order_data(inode)) {
			loff_t start_byte =
				(loff_t)map->m_lblk << inode->i_blkbits;
			loff_t length = (loff_t)map->m_len << inode->i_blkbits;

			if (flags & EXT4_GET_BLOCKS_IO_SUBMIT)
				ret = ext4_jbd2_inode_add_wait(handle, inode,
						start_byte, length);
			else
				ret = ext4_jbd2_inode_add_write(handle, inode,
						start_byte, length);
			if (ret)
				return ret;
		}
	}
	if (retval > 0 && (map->m_flags & EXT4_MAP_UNWRITTEN ||
				map->m_flags & EXT4_MAP_MAPPED))
		ext4_fc_track_range(handle, inode, map->m_lblk,
					map->m_lblk + map->m_len - 1);
	if (retval < 0)
		ext_debug(inode, "failed with err %d\n", retval);
	return retval;
}
```
**事务处理和文件系统一致性:**
- `if (map->m_flags & EXT4_MAP_NEW && !(map->m_flags & EXT4_MAP_UNWRITTEN) && !(flags & EXT4_GET_BLOCKS_ZERO) && !ext4_is_quota_file(inode) && ext4_should_order_data(inode))`:  条件判断，决定是否需要将 inode 添加到 JBD2 (Journaling Block Device layer 2) 的 ordered data list 中，以保证数据一致性。
    - `map->m_flags & EXT4_MAP_NEW`:  表示分配了新的块。
    - `!(map->m_flags & EXT4_MAP_UNWRITTEN)`:  表示不是未写入 extent。
    - `!(flags & EXT4_GET_BLOCKS_ZERO)`:  表示不需要零填充。
    - `!ext4_is_quota_file(inode)`:  不是 quota 文件。
    - `ext4_should_order_data(inode)`:  inode 应该使用 ordered data journaling 模式。
    - 如果所有条件都满足，表示新分配的块需要保证数据一致性，需要添加到 JBD2 的 ordered data list 中。
- `loff_t start_byte = (loff_t)map->m_lblk << inode->i_blkbits; loff_t length = (loff_t)map->m_len << inode->i_blkbits;`:  计算起始字节偏移量和长度。
- `if (flags & EXT4_GET_BLOCKS_IO_SUBMIT) ret = ext4_jbd2_inode_add_wait(handle, inode, start_byte, length); else ret = ext4_jbd2_inode_add_write(handle, inode, start_byte, length);`:  根据 `EXT4_GET_BLOCKS_IO_SUBMIT` 标志选择不同的 JBD2 函数将 inode 添加到 ordered data list 中。
    - `ext4_jbd2_inode_add_wait`:  同步添加，等待事务提交完成。
    - `ext4_jbd2_inode_add_write`:  异步添加，不等待事务提交。
- `if (ret) return ret;`:  如果 JBD2 添加操作失败，返回错误码 `ret`。
- `if (retval > 0 && (map->m_flags & EXT4_MAP_UNWRITTEN || map->m_flags & EXT4_MAP_MAPPED)) ext4_fc_track_range(handle, inode, map->m_lblk, map->m_lblk + map->m_len - 1);`:  如果成功映射或分配了块（包括 unwritten extent），则调用 `ext4_fc_track_range` 函数跟踪 extent 的范围，用于文件系统 features compatibility (fc) 管理。
- `if (retval < 0) ext_debug(inode, "failed with err %d\n", retval);`:  如果 `retval` 小于 0，表示操作失败，输出调试信息。
- `return retval;`:  返回最终结果 `retval`。

**总结事务处理和文件系统一致性部分:**

这部分代码处理了新分配块的事务一致性问题，确保在 ordered data journaling 模式下，新写入的数据在事务提交后才能对外可见。同时，还跟踪了 extent 的范围，用于文件系统特性兼容性管理。

**`ext4_map_blocks` 函数在 `ext4_mpage_readpages` 中的角色:**

在 `ext4_mpage_readpages` 函数中，`ext4_map_blocks` 被用来获取每个 folio 页面的物理块映射。`ext4_mpage_readpages` 循环处理多个页面，对于每个页面，它会调用 `ext4_map_blocks` 来查找或分配物理块。

- **查询映射:**  `ext4_mpage_readpages` 主要用于读取操作，通常情况下，它会调用 `ext4_map_blocks` 来查询已有的块映射，而不是分配新块（除非是预读到文件尾部需要扩展文件）。在 `ext4_mpage_readpages` 的代码中，`flags` 参数通常设置为 0，表示只进行查询操作，不创建块。
- **`ext4_map_blocks` 结构体的使用:**
    - `ext4_mpage_readpages` 初始化 `ext4_map_blocks` 结构体 `map`，设置 `m_lblk` 为要读取的逻辑块号，`m_len` 为请求的长度。
    - `ext4_map_blocks` 函数根据 `m_lblk` 和 `m_len` 在 inode 的 extent tree 或间接块映射结构中查找对应的物理块映射。
    - `ext4_map_blocks` 将查找结果（物理块号 `m_pblk`，标志 `m_flags`，实际映射长度 `m_len`）写回 `map` 结构体。
    - `ext4_mpage_readpages` 根据 `map` 结构体中的结果，构建 `bio` 请求，读取对应的物理块数据到 folio 页面。

**Extent vs. Indirect Mapping:**

`ext4_map_blocks` 函数本身并不直接区分 extent 和 indirect mapping 的处理逻辑。 实际的区分和处理是在其调用的子函数中完成的，例如 `ext4_es_lookup_extent`, `ext4_map_query_blocks`, 和 `ext4_map_create_blocks`。

- **Extent-based 文件:**  对于 extent-based 文件，`ext4_es_lookup_extent` 会在 extent status tree 中查找 extent 信息。`ext4_map_query_blocks` 和 `ext4_map_create_blocks` 会操作 inode 的 extent tree 来查询和分配 extent。
- **Indirect-based 文件:**  对于 indirect-based 文件（传统的文件类型，不使用 extent），`ext4_es_lookup_extent` 可能不会找到 extent 信息。`ext4_map_query_blocks` 和 `ext4_map_create_blocks` 会使用传统的间接块映射方式，遍历 inode 的 i_block 数组和间接块，进行块映射查询和分配。

**总结 `ext4_map_blocks` 函数:**

`ext4_map_blocks` 是 ext4 文件系统中核心的块映射函数，它负责根据逻辑块号查找或分配物理块。它首先尝试在 extent status tree 中快速查找，如果找不到，则进行更全面的查询（`ext4_map_query_blocks`）或块分配（`ext4_map_create_blocks`）。函数内部会根据 inode 的类型（extent-based 或 indirect-based）选择不同的映射方式。在 `ext4_mpage_readpages` 中，`ext4_map_blocks` 主要用于查询已有的块映射，为后续的 I/O 操作提供物理块信息。 函数还处理了事务一致性、文件系统特性兼容性等问题，保证文件系统的可靠性和性能。

希望这个更深入的分析能够帮助你理解 `ext4_map_blocks` 函数的复杂性和重要性。 如果还有其他问题，请随时提出。

