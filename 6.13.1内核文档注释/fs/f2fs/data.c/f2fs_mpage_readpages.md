**相关函数**
*   [read_pages](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/readahead.c/read_pages.md)
*	[f2fs_read_multipages](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/data.c/f2fs_read_multi_pages.md)
** `f2fs_mpage_readpages`**

 当然！这是我上次回复的完整中文翻译，一字不差：

## 函数概述

`f2fs_mpage_readpages` 函数是 F2FS (Flash-Friendly File System，闪存友好文件系统) 的核心组件，用于将磁盘数据读取到内存页中。它旨在处理单页读取和预读请求，并且它已经从更通用的 `mpage_readpages` 函数调整而来，以适应 F2FS 的特定需求。 F2FS 中的一个关键定制是，默认情况下块大小通常等于页大小，这简化了与传统文件系统相比的一些块映射方面，在传统文件系统中，块大小可能小于页大小。

如果启用了 `CONFIG_F2FS_FS_COMPRESSION` 选项，该函数还包含处理文件压缩的逻辑。

## 非压缩逻辑（简化解释）

首先，让我们想象一下文件压缩**禁用** (`#ifndef CONFIG_F2FS_FS_COMPRESSION`) 的情况。在这种情况下，代码变得简单得多，并且专注于直接从磁盘读取页面。

```c
static int f2fs_mpage_readpages(struct inode *inode,
		struct readahead_control *rac, struct folio *folio)
{
	struct bio *bio = NULL; // bio 用于 I/O 操作
	sector_t last_block_in_bio = 0; // 跟踪 bio 中的最后一个块
	struct f2fs_map_blocks map; // 块映射信息
	unsigned nr_pages = rac ? readahead_count(rac) : 1; // 要读取的页数
	unsigned max_nr_pages = nr_pages; // 存储初始页数
	int ret = 0; // 返回值

	map.m_pblk = 0; // 初始化映射结构
	map.m_lblk = 0;
	map.m_len = 0;
	map.m_flags = 0;
	map.m_next_pgofs = NULL;
	map.m_next_extent = NULL;
	map.m_seg_type = NO_CHECK_TYPE;
	map.m_may_create = false;

	for (; nr_pages; nr_pages--) { // 循环读取每个页面
		if (rac) { // 如果预读处于活动状态
			folio = readahead_folio(rac); // 从预读中获取下一个 folio
			prefetchw(&folio->flags); // 预取 folio 标志 (优化)
		}

		// --- 在非压缩情况下跳过压缩检查 ---

		ret = f2fs_read_single_page(inode, folio, max_nr_pages, &map,
					&bio, &last_block_in_bio, rac); // 读取单个页面
		if (ret) { // 错误处理
			folio_zero_segment(folio, 0, folio_size(folio)); // 在错误时将 folio 清零
			folio_unlock(folio); // 解锁 folio
		}

		// --- 跳过与压缩相关的 "next_page" 标签 ---
	}
	if (bio) // 如果创建了 bio (添加了页面)
		f2fs_submit_read_bio(F2FS_I_SB(inode), bio, DATA); // 提交读取 bio
	return ret; // 返回状态
}
```

让我们逐步了解这个简化版本：

1.  **初始化：**
    *   `struct bio *bio = NULL;`:  `bio` 结构初始化为 `NULL`。 Linux 中的 `bio` (Block I/O，块 I/O) 结构表示块 I/O 操作。我们将使用它将多个页面读取操作组合成单个 I/O 请求，以提高效率。
    *   `sector_t last_block_in_bio = 0;`: 此变量跟踪添加到当前 `bio` 的最后一个磁盘块号。这对于将连续块读取合并到单个 `bio` 中非常重要。
    *   `struct f2fs_map_blocks map;`: 此结构用于存储逻辑块地址（在文件内）和物理块地址（在磁盘上）之间的映射。它将由确定给定页面的数据在磁盘上的位置的函数填充。
    *   `unsigned nr_pages = rac ? readahead_count(rac) : 1;`:  这确定要读取多少页。如果提供了 `rac` (预读控制结构)（意味着预读处于活动状态），它会从 `readahead_count(rac)` 获取要读取的页数。否则，它默认为仅读取一页。
    *   `unsigned max_nr_pages = nr_pages;`:  这存储了 `nr_pages` 的初始值。它稍后会被使用，可能在循环内调用的函数中，可能是为了上下文或限制。
    *   `int ret = 0;`:  此变量将存储函数的返回状态。 `0` 通常表示成功，负值表示错误。
    *   `map` 结构使用默认值初始化。 `m_seg_type` 的 `NO_CHECK_TYPE` 表明我们在此映射操作期间不强制执行特定的段类型约束。 `m_may_create = false` 表明此映射操作用于读取现有块，而不是分配新块。

2.  **页面读取循环：** `for (; nr_pages; nr_pages--)`
    *   此循环迭代的次数与我们需要读取的页数 (`nr_pages`) 相同。在每次迭代中，它处理一个页面。

3.  **预读处理：** `if (rac) { ... }`
    *   `if (rac)`: 此条件检查是否提供了 `readahead_control` 结构 (`rac`)。 如果是，则表示为此文件启用了预读。
    *   `folio = readahead_folio(rac);`:  `readahead_folio(rac)` 是一个函数，它获取预读机制已决定读取的下一个 `folio`（内存页，由 folio API 管理，类似于 `page` 但更现代）。 预读是一种性能优化技术，系统会预测哪些页面很快会被需要，并主动将它们读取到内存中，希望在应用程序实际请求它们时避免延迟。
    *   `prefetchw(&folio->flags);`: `prefetchw` 是一条 CPU 指令，它提示处理器将包含 `folio->flags` 的缓存行预取到缓存中。 这是一种性能优化，可以潜在地加速稍后对 folio 标志的访问。

4.  **`f2fs_read_single_page`：**
    *   `ret = f2fs_read_single_page(inode, folio, max_nr_pages, &map, &bio, &last_block_in_bio, rac);`: 这是用于读取单个页面的核心函数调用。 让我们分解一下参数：
        *   `inode`: 要读取文件的 inode。 inode 包含有关文件的元数据，包括其在磁盘上的位置。
        *   `folio`: 将在其中读取数据的内存页 (folio)。
        *   `max_nr_pages`: 请求的初始页数（为了上下文而传递下来）。
        *   `&map`: 指向 `f2fs_map_blocks` 结构的指针。 `f2fs_read_single_page` 可能会使用此结构来获取与文件中逻辑页面对应的磁盘上的物理块地址。
        *   `&bio`: 指向 `bio` 结构的指针。 `f2fs_read_single_page` 可能会将当前页面的读取操作添加到此 `bio`。 如果 `bio` 最初为 `NULL`，则 `f2fs_read_single_page` 可能会分配并初始化它。
        *   `&last_block_in_bio`: 指向 `last_block_in_bio` 的指针。 `f2fs_read_single_page` 将更新此值以反映添加到 `bio` 的最后一个块。 这有助于合并连续读取。
        *   `rac`: 预读控制结构（传递下去以便在 `f2fs_read_single_page` 中潜在地使用）。

5.  **错误处理：** `if (ret) { ... }`
    *   `if (ret)`: 检查 `f2fs_read_single_page` 是否返回错误（非零值）。
    *   `folio_zero_segment(folio, 0, folio_size(folio));`: 如果读取页面时发生错误，则此行将 folio 的内容清零。 这对于防止进程使用可能已损坏或未初始化的数据非常重要。
    *   `folio_unlock(folio);`:  解锁 folio。 Folio 通常被锁定以防止并发访问。 如果发生错误，解锁 folio 以允许其他操作继续进行至关重要。

6.  **提交 BIO：** `if (bio) f2fs_submit_read_bio(F2FS_I_SB(inode), bio, DATA);`
    *   `if (bio)`: 循环结束后，这将检查 `bio` 是否不为 `NULL`。 如果不为 `NULL`，则表示在一个或多个页面在循环期间已成功添加到 `bio` 结构中。
    *   `f2fs_submit_read_bio(F2FS_I_SB(inode), bio, DATA);`: 此函数将 `bio` 提交给 I/O 子系统以实际执行磁盘读取操作。
        *   `F2FS_I_SB(inode)`: 获取与 inode 关联的 F2FS 文件系统的超级块。 超级块包含 I/O 操作所需的文件系统级信息。
        *   `bio`: 包含读取请求的 `bio` 结构。
        *   `DATA`: 这可能是一个枚举值或常量，指示 I/O 操作的类型（在本例中为读取数据块）。

7.  **返回值：** `return ret;`
    *   函数返回 `ret` 的值。 如果循环期间未发生任何错误，则 `ret` 将保持为 `0`，表示成功。 否则，它将返回来自 `f2fs_read_single_page` 的错误代码。

**本质上，非压缩逻辑执行以下操作：**

*   确定要读取多少页（基于预读或默认为 1）。
*   对于每个页面：
    *   获取一个 folio（内存页）。
    *   调用 `f2fs_read_single_page` 以从磁盘读取页面（这处理块映射并可能将读取添加到 `bio`）。
    *   通过将 folio 清零并解锁来处理错误。
*   提交累积的 `bio`（如果有）到 I/O 子系统。

## 压缩逻辑（增加复杂性）

现在，让我们添加回压缩逻辑 (`#ifdef CONFIG_F2FS_FS_COMPRESSION ... #endif`)。 这为函数增加了显著的复杂性。

```c
static int f2fs_mpage_readpages(struct inode *inode,
		struct readahead_control *rac, struct folio *folio)
{
	// ... (与之前相同的初始化) ...
#ifdef CONFIG_F2FS_FS_COMPRESSION
	struct compress_ctx cc = { // 压缩上下文
		.inode = inode,
		.log_cluster_size = F2FS_I(inode)->i_log_cluster_size,
		.cluster_size = F2FS_I(inode)->i_cluster_size,
		.cluster_idx = NULL_CLUSTER,
		.rpages = NULL,
		.cpages = NULL,
		.nr_rpages = 0,
		.nr_cpages = 0,
	};
	pgoff_t nc_cluster_idx = NULL_CLUSTER; // 非压缩簇索引
	pgoff_t index; // Folio 索引
#endif
	// ... (与之前相同的 nr_pages 循环) ...
#ifdef CONFIG_F2FS_FS_COMPRESSION
		index = folio_index(folio); // 获取 folio 索引 注意这个是从内存中拿到的folio的索引

		if (!f2fs_compressed_file(inode)) // 检查文件是否已压缩
			goto read_single_page; // 如果未压缩，则作为单页读取

		/* 这个判断虽然在前但是一般是后触发的
		一旦新的page index终于无法和前面的簇合并
		那之前初始化好的ctx如果存在剩余的压缩页面，提交它们
		 */
		if (!f2fs_cluster_can_merge_page(&cc, index)) { // 无法合并到当前簇？
			ret = f2fs_read_multi_pages(&cc, &bio, // 从压缩上下文中读取多页
						max_nr_pages,
						&last_block_in_bio,
						rac, false);
			f2fs_destroy_compress_ctx(&cc, false); 
			/* 销毁压缩上下文 并且这个上下文不会再次被复用。所以这行函数调用结束之后
			cc.cluster_idx一定变为null_cluster*/
			if (ret)
				goto set_error_page; // 错误处理
		}
		if (cc.cluster_idx == NULL_CLUSTER) { // 没有当前的压缩簇？
			if (nc_cluster_idx == index >> cc.log_cluster_size) // 相同的非压缩簇？
				goto read_single_page; // 作为单页读取

			ret = f2fs_is_compressed_cluster(inode, index); // 检查簇是否已压缩
			if (ret < 0)
				goto set_error_page; // 错误处理
			else if (!ret) { // 不是压缩簇
				nc_cluster_idx =
					index >> cc.log_cluster_size; // 更新非压缩簇索引
				goto read_single_page; // 作为单页读取
			}

			nc_cluster_idx = NULL_CLUSTER; // 重置非压缩簇索引
		}
		ret = f2fs_init_compress_ctx(&cc); // 为新簇初始化压缩上下文 具体来来说是为cc的rpages指针分配内存
										   // 如果已经分配过了就不再分配了。
		if (ret)
			goto set_error_page; // 错误处理

		f2fs_compress_ctx_add_page(&cc, folio); // 将页面添加到压缩上下文。同属于一个cluster的page会全部因为
												// f2fs_cluster_can_merge_page为true 直接跳过read_multipages
												// 转到add_page直接添加。

		goto next_page; // 转到下一个页面处理
read_single_page: // 用于读取单页的标签（非压缩路径）
#endif

		ret = f2fs_read_single_page(inode, folio, max_nr_pages, &map,
					&bio, &last_block_in_bio, rac); // 读取单页（与之前相同）
		if (ret) {
#ifdef CONFIG_F2FS_FS_COMPRESSION
set_error_page: // 错误处理标签（由两个路径使用）
#endif
			folio_zero_segment(folio, 0, folio_size(folio)); // 将 folio 清零
			folio_unlock(folio); // 解锁 folio
		}
#ifdef CONFIG_F2FS_FS_COMPRESSION
next_page: // 用于下一个页面处理的标签（在压缩路径中使用）
#endif

#ifdef CONFIG_F2FS_FS_COMPRESSION
		if (f2fs_compressed_file(inode)) { // 再次检查文件是否已压缩（用于最后一个页面处理）
			/* 最后一个页面 */
			if (nr_pages == 1 && !f2fs_cluster_is_empty(&cc)) { // 最后一个页面且压缩上下文不为空？
				ret = f2fs_read_multi_pages(&cc, &bio, // 读取剩余的多页
							max_nr_pages,
							&last_block_in_bio,
							rac, false);
				f2fs_destroy_compress_ctx(&cc, false); // 销毁压缩上下文
			}
		}
#endif
	}
	// ... (与之前相同的 bio 提交和返回) ...
}
```

以下是压缩逻辑的集成方式：

1.  **压缩上下文 (`struct compress_ctx cc`)**：
    *   引入了 `struct compress_ctx` 来管理与读取压缩数据相关的状态。 它包含：
        *   `inode`: 文件的 inode。
        *   `log_cluster_size`, `cluster_size`:  有关用于压缩的簇大小的信息。 F2FS 使用簇作为压缩单位。
        *   `cluster_idx`:  当前正在处理的压缩簇的索引。 `NULL_CLUSTER` 可能表示当前未处理任何簇。
        *   `rpages`, `cpages`: 可能是列表或数组，用于存储与当前簇关联的原始（未压缩）页面和压缩页面。
        *   `nr_rpages`, `nr_cpages`: 原始页面和压缩页面的计数器。

2.  **非压缩簇索引 (`nc_cluster_idx`)**：
    *   `pgoff_t nc_cluster_idx = NULL_CLUSTER;`: 此变量似乎跟踪上次遇到的*非压缩*簇的索引。 这用于优化连续页面属于同一非压缩簇的情况。

3.  **文件压缩检查：** `if (!f2fs_compressed_file(inode)) goto read_single_page;`
    *   在每个页面迭代的开始处（在 `#ifdef CONFIG_F2FS_FS_COMPRESSION` 块内），它使用 `f2fs_compressed_file(inode)` 检查与 `inode` 关联的文件是否实际已压缩。
    *   如果文件*未*压缩，它会立即跳转到 `read_single_page` 标签。 这是一个至关重要的优化：如果文件未压缩，我们绕过所有与压缩相关的逻辑，并将其作为常规单页读取（如非压缩部分所述）。

4.  **簇合并检查：** `if (!f2fs_cluster_can_merge_page(&cc, index)) { ... }`
    *   `f2fs_cluster_can_merge_page(&cc, index)`: 此函数可能检查当前页面（由其 `index` 指示，它是文件中的页面偏移量）是否可以合并到 `compress_ctx` (`cc`) 中正在处理的*当前*压缩簇中。
    *   如果它返回 `false`，则表示当前页面无法合并到当前簇中。 这可能是因为：
        *   当前压缩簇已满。
        *   当前页面属于*不同的*压缩簇。
        *   我们正在从压缩簇过渡到文件中的非压缩区域。
    *   **如果无法合并：**
        *   `f2fs_read_multi_pages(&cc, &bio, ...)`:  调用此函数以读取已在 `compress_ctx` (`cc`) 中累积的*多页*压缩簇。 它从 `cc` 中获取页面，确定如何从磁盘读取它们（可能涉及元数据查找以查找压缩块），并将读取操作添加到 `bio`。
        *   `f2fs_destroy_compress_ctx(&cc, false);`: 读取多页簇后，压缩上下文 `cc` 将被销毁（重置或释放），因为我们开始处理新的簇（或可能是非压缩页面）。

5.  **没有当前的压缩簇检查：** `if (cc.cluster_idx == NULL_CLUSTER) { ... }`
    *   此条件检查 `cc.cluster_idx` 是否为 `NULL_CLUSTER`。 这意味着我们当前未处理压缩簇。
    *   **非压缩簇优化：**
        *   `if (nc_cluster_idx == index >> cc.log_cluster_size) goto read_single_page;`: 这是另一个优化。 它检查当前页面是否与*前一个*页面属于*相同的*非压缩簇。 `index >> cc.log_cluster_size` 计算当前页面的簇索引。 如果它与 `nc_cluster_idx` 匹配，则表示我们仍在相同的非压缩簇中，因此我们可以将此页面作为单页读取（使用 `goto read_single_page`）。
    *   **检查当前簇是否已压缩：**
        *   `ret = f2fs_is_compressed_cluster(inode, index);`: 如果我们与之前的非压缩簇不在同一个簇中，我们需要确定*当前*簇（包含当前页面）是否已压缩。 `f2fs_is_compressed_cluster` 可能会检查与文件或簇关联的元数据，以确定其压缩状态。
        *   如果 `f2fs_is_compressed_cluster` 返回：
            *   `< 0`: 错误，转到 `set_error_page`。
            *   `0`: 不是压缩簇。
                *   `nc_cluster_idx = index >> cc.log_cluster_size;`: 更新 `nc_cluster_idx` 以记住此非压缩簇的索引。
                *   `goto read_single_page;`: 将此页面作为单个非压缩页面读取。
            *   `> 0`（或某些其他非零值）：它是压缩簇。
                *   `nc_cluster_idx = NULL_CLUSTER;`: 重置 `nc_cluster_idx`，因为我们现在正在进入压缩簇。

6.  **为新簇初始化压缩上下文：** `ret = f2fs_init_compress_ctx(&cc);`
    *   如果我们已确定当前簇已压缩，并且我们尚未处理压缩簇，则调用 `f2fs_init_compress_ctx(&cc)`。 此函数可能会初始化 `compress_ctx` (`cc`) 以开始累积新压缩簇的页面。

7.  **将页面添加到压缩上下文：** `f2fs_compress_ctx_add_page(&cc, folio);`
    *   `f2fs_compress_ctx_add_page(&cc, folio)`: 此函数将当前 `folio` 添加到 `compress_ctx` (`cc`)。 这意味着我们正在 `cc` 中累积属于当前压缩簇的页面。 当我们稍后读取多页压缩簇时，将一起处理这些页面。

8.  **`goto next_page;`**: 将页面添加到压缩上下文后，我们跳转到 `next_page` 标签以继续处理预读或多页请求中的下一个页面。

9.  **`read_single_page:` 标签**： 当以下情况发生时，会到达此标签：
    *   文件未压缩。
    *   我们确定当前簇未压缩。
    *   我们与前一个页面在相同的非压缩簇中（优化）。
    *   在所有这些情况下，我们都回退到使用 `f2fs_read_single_page` 的非压缩单页读取路径，如非压缩部分所述。

10. **`set_error_page:` 标签**： 这是错误处理标签，如果在任何压缩相关的检查或操作失败时到达。 它与非压缩路径中的错误处理相同：将 folio 清零并解锁。

11. **`next_page:` 标签**： 此标签只是压缩路径中代码流程的标记，用于继续到页面循环的下一次迭代。

12. **最后一个页面处理（压缩文件）：**
    *   `if (f2fs_compressed_file(inode)) { ... if (nr_pages == 1 && !f2fs_cluster_is_empty(&cc)) { ... } }`
    *   在主页面循环结束后，有一个针对压缩文件的特殊检查。
    *   `if (nr_pages == 1 && !f2fs_cluster_is_empty(&cc))`: 此条件检查是否：
        *   `nr_pages == 1`: 我们正在处理请求读取的*最后一个*页面（因为循环会递减 `nr_pages`）。
        *   `!f2fs_cluster_is_empty(&cc)`: `compress_ctx` (`cc`) *不*为空。 这意味着 `cc` 中仍然累积了属于尚未读取的压缩簇的页面。 如果读取请求在压缩簇的中间结束，则可能会发生这种情况。
    *   **如果两个条件都为真（最后一个页面和挂起的压缩页面）：**
        *   `f2fs_read_multi_pages(&cc, &bio, ...)`: 我们需要从 `compress_ctx` 读取剩余的多页压缩簇，以确保我们读取簇的所有数据，即使原始读取请求的页面较少。
        *   `f2fs_destroy_compress_ctx(&cc, false);`: 在读取最后一个簇后销毁压缩上下文。

**压缩逻辑总结：**

*   **管理压缩簇：** 代码旨在以压缩簇为单位读取数据。 它在 `compress_ctx` 中累积属于簇的页面。
*   **压缩簇的多页读取：**  它使用 `f2fs_read_multi_pages` 一次读取整个压缩簇，这可能比读取单个压缩页面更有效。
*   **处理非压缩文件和簇：** 它有效地处理文件或文件的某些部分未压缩的情况，回退到使用 `f2fs_read_single_page` 的单页读取。
*   **优化：** 包括优化以避免不必要的压缩检查，并有效地处理同一非压缩簇内的连续页面。
*   **最后一个页面处理：**  确保即使读取请求在簇中间结束，如果需要，也会读取整个最后一个压缩簇。

**本质上，压缩逻辑在基本页面读取之上添加了一个基于簇的处理层。 它智能地在非压缩数据的单页读取和压缩簇的多页读取之间切换，旨在在两种情况下都实现效率。**

如果您想更深入地了解函数的任何特定部分或压缩逻辑，请告诉我！

我们现在尝试改造一下`f2fs_mpage_readpages`函数使得其能处理任意阶folio:
