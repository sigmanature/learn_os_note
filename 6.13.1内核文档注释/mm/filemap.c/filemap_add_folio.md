 好的，让我们来逐行解析 `filemap_add_folio` 和 `__filemap_add_folio` 这两个函数，并解答你的问题：`filemap_add_folio` 是否会将 folio 和对应的 index 作为键值对一起放到 page cache 中。

**1. `filemap_add_folio` 函数解析**

```c
int filemap_add_folio(struct address_space *mapping, struct folio *folio,
				pgoff_t index, gfp_t gfp)
{
	void *shadow = NULL;
	int ret;

	ret = mem_cgroup_charge(folio, NULL, gfp);
	if (ret)
		return ret;

	__folio_set_locked(folio);
	ret = __filemap_add_folio(mapping, folio, index, gfp, &shadow);
	if (unlikely(ret)) {
		mem_cgroup_uncharge(folio);
		__folio_clear_locked(folio);
	} else {
		/*
		 * The folio might have been evicted from cache only
		 * recently, in which case it should be activated like
		 * any other repeatedly accessed folio.
		 * The exception is folios getting rewritten; evicting other
		 * data from the working set, only to cache data that will
		 * get overwritten with something else, is a waste of memory.
		 */
		WARN_ON_ONCE(folio_test_active(folio));
		if (!(gfp & __GFP_WRITE) && shadow)
			workingset_refault(folio, shadow);
		folio_add_lru(folio);
	}
	return ret;
}
```

*   **`int filemap_add_folio(struct address_space *mapping, struct folio *folio, pgoff_t index, gfp_t gfp)`**: 函数定义。
    *   `int`: 返回值类型为整型，表示操作是否成功。0 表示成功，负值表示错误。
    *   参数：
        *   `struct address_space *mapping`:  文件相关的地址空间。
        *   `struct folio *folio`:  要添加到页缓存的 folio。
        *   `pgoff_t index`:  folio 在地址空间中的索引（页偏移）。
        *   `gfp_t gfp`:  内存分配标志。
*   **`void *shadow = NULL;`**:  声明一个 `void *` 类型的指针 `shadow` 并初始化为 `NULL`。 `shadow` 变量将在后续的 `__filemap_add_folio` 函数中被使用，用于处理可能的 shadow 项（例如，用于 workingset 管理）。
*   **`int ret;`**:  声明一个整型变量 `ret`，用于存储函数调用的返回值。
*   **`ret = mem_cgroup_charge(folio, NULL, gfp);`**:  进行内存控制组 (memcg) 的收费 (charge) 操作。
    *   `mem_cgroup_charge(folio, NULL, gfp)`:  尝试向 folio 关联的内存控制组收取内存费用。如果内存控制组有资源限制，并且当前操作会导致超出限制，则此函数可能返回错误。
    *   `folio`:  要收费的 folio。
    *   `NULL`:  通常用于传递父 memcg 上下文，这里为 `NULL` 表示使用当前的 memcg 上下文。
    *   `gfp`:  内存分配标志，可能影响收费行为。
    *   `ret = ...`:  将 `mem_cgroup_charge` 的返回值赋给 `ret`。如果收费失败，`ret` 将为非零值。
*   **`if (ret) return ret;`**:  检查 `mem_cgroup_charge` 的返回值 `ret`。如果 `ret` 非零（表示收费失败），则直接返回 `ret`，表示 `filemap_add_folio` 操作失败。
*   **`__folio_set_locked(folio);`**:  设置 folio 的锁状态为 locked。
    *   `__folio_set_locked(folio)`:  这是一个底层函数，用于设置 folio 的锁标志。在将 folio 添加到页缓存之前，通常需要先锁定 folio，以防止并发访问和修改。
*   **`ret = __filemap_add_folio(mapping, folio, index, gfp, &shadow);`**:  调用核心的 `__filemap_add_folio` 函数来实际执行将 folio 添加到页缓存的操作。
    *   `__filemap_add_folio(mapping, folio, index, gfp, &shadow)`:  执行将 folio 添加到地址空间 `mapping` 的 `i_pages` xarray 中的核心逻辑。
    *   `&shadow`:  传递 `shadow` 变量的地址，`__filemap_add_folio` 可能会修改 `shadow` 指针，用于返回可能存在的 shadow 项。
    *   `ret = ...`:  将 `__filemap_add_folio` 的返回值赋给 `ret`。
*   **`if (unlikely(ret))`**:  检查 `__filemap_add_folio` 的返回值 `ret`。 `unlikely(ret)` 是一个宏，用于提示编译器 `ret` 为真的可能性较低，用于优化分支预测。
    *   如果 `ret` 非零（表示 `__filemap_add_folio` 返回错误），则执行 `if` 代码块中的错误处理逻辑。
*   **`mem_cgroup_uncharge(folio);`**:  如果 `__filemap_add_folio` 失败，需要撤销之前成功的内存控制组收费操作。
    *   `mem_cgroup_uncharge(folio)`:  撤销之前对 `folio` 的内存控制组收费。
*   **`__folio_clear_locked(folio);`**:  清除之前设置的 folio 锁状态。
    *   `__folio_clear_locked(folio)`:  解锁 folio。
*   **`else { ... }`**:  如果 `__filemap_add_folio` 返回成功（`ret` 为 0），则执行 `else` 代码块中的成功处理逻辑。
*   **`/* ... 注释 ... */`**:  注释块解释了接下来的代码逻辑：如果 folio 最近才从缓存中驱逐出去，应该像其他重复访问的 folio 一样激活它。但对于将被重写 (rewritten) 的 folio，则不应该激活，因为那样会浪费内存。
*   **`WARN_ON_ONCE(folio_test_active(folio));`**:  检查 folio 是否已经被标记为 active 状态。 `WARN_ON_ONCE` 是一个宏，如果条件为真，则打印一次警告信息。这里检查的目的是为了确保新添加的 folio 不应该已经是 active 状态（active 状态通常表示 folio 正在被使用或最近被使用）。
    *   `folio_test_active(folio)`:  检查 folio 是否设置了 active 标志。
*   **`if (!(gfp & __GFP_WRITE) && shadow)`**:  条件判断是否需要进行 workingset refault 操作。
    *   `!(gfp & __GFP_WRITE)`:  检查内存分配标志 `gfp` 中是否没有设置 `__GFP_WRITE` 标志。 `__GFP_WRITE` 通常表示分配内存用于写入操作。 如果没有设置 `__GFP_WRITE`，则表示当前操作是读取操作。
    *   `shadow`:  检查 `shadow` 指针是否非空。 `shadow` 指针可能在 `__filemap_add_folio` 中被设置为指向一个 shadow 项。
    *   `&&`:  逻辑与运算符。 只有当 `gfp` 中没有 `__GFP_WRITE` 标志 *并且* `shadow` 指针非空时，条件才为真。
*   **`workingset_refault(folio, shadow);`**:  如果满足上述条件，则调用 `workingset_refault` 函数进行 workingset refault 操作。
    *   `workingset_refault(folio, shadow)`:  用于更新 folio 的 workingset 状态。 Workingset 是一种内存管理机制，用于跟踪哪些页面是 "working set" 的一部分，以便更好地进行内存回收和页面置换。 Refault 操作通常发生在页面被重新访问时，用于更新其在 workingset 中的状态。
*   **`folio_add_lru(folio);`**:  将 folio 添加到 LRU (Least Recently Used) 链表中。
    *   `folio_add_lru(folio)`:  将 folio 添加到全局或 per-lruvec 的 LRU 链表中。 LRU 链表用于页面置换算法，用于跟踪哪些页面最近最少使用，以便在内存不足时优先回收这些页面。 将 folio 添加到 LRU 链表意味着它现在可以被纳入页面置换的考虑范围。
*   **`return ret;`**:  返回函数执行结果 `ret`。 对于成功的情况，`ret` 应该为 0。

**总结 `filemap_add_folio`:**

`filemap_add_folio` 函数是向页缓存添加 folio 的上层接口。它主要负责以下步骤：

1.  **内存控制组收费**:  调用 `mem_cgroup_charge` 进行内存控制组收费。
2.  **锁定 Folio**:  调用 `__folio_set_locked` 锁定 folio。
3.  **核心添加操作**:  调用 `__filemap_add_folio` 执行实际的添加 folio 到页缓存的操作。
4.  **错误处理**:  如果 `__filemap_add_folio` 失败，则撤销内存控制组收费，解锁 folio。
5.  **成功处理**:  如果 `__filemap_add_folio` 成功，则进行 workingset refault (如果条件满足)，并将 folio 添加到 LRU 链表。
6.  **返回结果**:  返回操作结果。

**2. `__filemap_add_folio` 函数解析**

```c
noinline int __filemap_add_folio(struct address_space *mapping,
		struct folio *folio, pgoff_t index, gfp_t gfp, void **shadowp)
{
	XA_STATE(xas, &mapping->i_pages, index);
	void *alloced_shadow = NULL;
	int alloced_order = 0;
	bool huge;
	long nr;

	VM_BUG_ON_FOLIO(!folio_test_locked(folio), folio);
	VM_BUG_ON_FOLIO(folio_test_swapbacked(folio), folio);
	VM_BUG_ON_FOLIO(folio_order(folio) < mapping_min_folio_order(mapping),
			folio);
	mapping_set_update(&xas, mapping);

	VM_BUG_ON_FOLIO(index & (folio_nr_pages(folio) - 1), folio);
	xas_set_order(&xas, index, folio_order(folio));
	huge = folio_test_hugetlb(folio);
	nr = folio_nr_pages(folio);

	gfp &= GFP_RECLAIM_MASK;
	folio_ref_add(folio, nr);
	folio->mapping = mapping;
	folio->index = xas.xa_index;

	for (;;) {
		int order = -1, split_order = 0;
		void *entry, *old = NULL;

		xas_lock_irq(&xas);
		xas_for_each_conflict(&xas, entry) {
			old = entry;
			if (!xa_is_value(entry)) {
				xas_set_err(&xas, -EEXIST);
				goto unlock;
			}
			/*
			 * If a larger entry exists,
			 * it will be the first and only entry iterated.
			 */
			if (order == -1)
				order = xas_get_order(&xas);
		}

		/* entry may have changed before we re-acquire the lock */
		if (alloced_order && (old != alloced_shadow || order != alloced_order)) {
			xas_destroy(&xas);
			alloced_order = 0;
		}

		if (old) {
			if (order > 0 && order > folio_order(folio)) {
				/* How to handle large swap entries? */
				BUG_ON(shmem_mapping(mapping));
				if (!alloced_order) {
					split_order = order;
					goto unlock;
				}
				xas_split(&xas, old, order);
				xas_reset(&xas);
			}
			if (shadowp)
				*shadowp = old;
		}

		xas_store(&xas, folio);
		if (xas_error(&xas))
			goto unlock;

		mapping->nrpages += nr;

		/* hugetlb pages do not participate in page cache accounting */
		if (!huge) {
			__lruvec_stat_mod_folio(folio, NR_FILE_PAGES, nr);
			if (folio_test_pmd_mappable(folio))
				__lruvec_stat_mod_folio(folio,
						NR_FILE_THPS, nr);
		}

unlock:
		xas_unlock_irq(&xas);

		/* split needed, alloc here and retry. */
		if (split_order) {
			xas_split_alloc(&xas, old, split_order, gfp);
			if (xas_error(&xas))
				goto error;
			alloced_shadow = old;
			alloced_order = split_order;
			xas_reset(&xas);
			continue;
		}

		if (!xas_nomem(&xas, gfp))
			break;
	}

	if (xas_error(&xas))
		goto error;

	trace_mm_filemap_add_to_page_cache(folio);
	return 0;
error:
	folio->mapping = NULL;
	/* Leave page->index set: truncation relies upon it */
	folio_put_refs(folio, nr);
	return xas_error(&xas);
}
```

*   **`noinline int __filemap_add_folio(struct address_space *mapping, struct folio *folio, pgoff_t index, gfp_t gfp, void **shadowp)`**: 函数定义。
    *   `noinline`:  指示编译器不要内联此函数，即使它可能是一个小的函数。
    *   `int`: 返回值类型为整型。
    *   参数：与 `filemap_add_folio` 相同，加上 `void **shadowp`。
        *   `void **shadowp`:  一个指向 `void *` 指针的指针。 用于返回可能存在的 shadow 项。
*   **`XA_STATE(xas, &mapping->i_pages, index);`**:  初始化 XA_STATE 结构体 `xas`，用于操作地址空间 `mapping` 的 `i_pages` xarray，起始索引为 `index`。 **这里 `i_pages` 就是页缓存的 xarray 数据结构。**
*   **`void *alloced_shadow = NULL;`**:  声明并初始化 `alloced_shadow` 指针为 `NULL`。 用于跟踪分配的 shadow 项。
*   **`int alloced_order = 0;`**:  声明并初始化 `alloced_order` 为 0。 用于跟踪分配的 shadow 项的 order。
*   **`bool huge;`**:  声明布尔变量 `huge`，用于标记 folio 是否是 hugetlb 页。
*   **`long nr;`**:  声明长整型变量 `nr`，用于存储 folio 包含的页数。
*   **`VM_BUG_ON_FOLIO(!folio_test_locked(folio), folio);`**:  使用 `VM_BUG_ON_FOLIO` 宏进行断言检查，如果 `folio` 没有被锁定 (`!folio_test_locked(folio)` 为真)，则触发 BUG。 确保在添加 folio 之前 folio 必须被锁定。
*   **`VM_BUG_ON_FOLIO(folio_test_swapbacked(folio), folio);`**:  断言检查，如果 `folio` 是 swapbacked 类型 (`folio_test_swapbacked(folio)` 为真)，则触发 BUG。  `filemap_add_folio` 通常用于文件页缓存，不应该用于 swapbacked 页。
*   **`VM_BUG_ON_FOLIO(folio_order(folio) < mapping_min_folio_order(mapping), folio);`**:  断言检查，如果 `folio` 的 order 小于地址空间允许的最小 folio order，则触发 BUG。 确保 folio 的大小符合地址空间的最小 folio 大小要求。
*   **`mapping_set_update(&xas, mapping);`**:  设置 xarray 状态 `xas` 以便在更新 xarray 时更新地址空间 `mapping` 的相关信息（例如，radix tree 的版本号）。
*   **`VM_BUG_ON_FOLIO(index & (folio_nr_pages(folio) - 1), folio);`**:  断言检查，如果 `index` 没有对齐到 folio 大小的边界，则触发 BUG。 确保索引 `index` 是 folio 大小的整数倍。
*   **`xas_set_order(&xas, index, folio_order(folio));`**:  设置 xarray 状态 `xas` 的 order 信息，用于后续的 xarray 操作。
*   **`huge = folio_test_hugetlb(folio);`**:  检查 folio 是否是 hugetlb 页，并将结果赋值给 `huge` 变量。
*   **`nr = folio_nr_pages(folio);`**:  获取 folio 包含的页数，并赋值给 `nr` 变量。
*   **`gfp &= GFP_RECLAIM_MASK;`**:  屏蔽 `gfp` 中的非 reclaim 相关的标志，只保留 reclaim 相关的标志。 `GFP_RECLAIM_MASK` 是一个宏，用于提取 gfp 标志中的 reclaim 部分。
*   **`folio_ref_add(folio, nr);`**:  增加 folio 的引用计数，增加的数量为 `nr` (folio 包含的页数)。
*   **`folio->mapping = mapping;`**:  设置 folio 的 `mapping` 成员指向传入的地址空间 `mapping`。 **将 folio 与地址空间关联起来。**
*   **`folio->index = xas.xa_index;`**:  设置 folio 的 `index` 成员为 `xas.xa_index`，即传入的 `index`。 **将 folio 与索引关联起来。 这就是将 index 作为键的一部分存储在 folio 中的体现。**

*   **`for (;;) { ... }`**:  无限循环，用于处理 xarray 中的冲突和内存分配重试。
*   **`int order = -1, split_order = 0;`**:  在循环内部声明并初始化 `order` 和 `split_order` 变量。 `order` 用于存储冲突项的 order， `split_order` 用于标记是否需要 split 操作。
*   **`void *entry, *old = NULL;`**:  声明 `entry` 和 `old` 指针，用于在循环中处理 xarray 条目。
*   **`xas_lock_irq(&xas);`**:  获取 xarray 状态 `xas` 的自旋锁，并禁用中断。  **保护 xarray 的并发访问。**
*   **`xas_for_each_conflict(&xas, entry) { ... }`**:  遍历 xarray 中与当前要添加的 folio 索引范围冲突的现有条目。
    *   `xas_for_each_conflict(&xas, entry)`:  这是一个宏，用于遍历 xarray 中与当前 `xas` 状态表示的索引范围冲突的条目。 冲突可能发生在要添加的 folio 的索引范围与 xarray 中已存在的条目的索引范围重叠时。
    *   `entry`:  循环变量，指向每个冲突的 xarray 条目。
    *   **`old = entry;`**:  将当前冲突条目 `entry` 赋值给 `old` 变量，用于后续处理。
    *   **`if (!xa_is_value(entry)) { ... }`**:  检查冲突条目 `entry` 是否不是一个 value 类型的条目。
        *   `!xa_is_value(entry)`:  如果 `entry` 不是 value 类型，则表示冲突条目是一个实际的 folio 或其他类型的指针。
        *   **`xas_set_err(&xas, -EEXIST);`**:  设置 xarray 状态 `xas` 的错误码为 `-EEXIST` (File exists)。 表示要添加的 folio 的索引位置已经存在一个条目，添加操作失败。
        *   **`goto unlock;`**:  跳转到 `unlock` 标签，释放锁并退出循环。
    *   **`/* ... 注释 ... */`**:  注释解释了如果存在更大的条目，它将是第一个也是唯一一个被迭代的条目。
    *   **`if (order == -1) order = xas_get_order(&xas);`**:  如果 `order` 变量还是初始值 -1，则从冲突条目 `entry` 中获取其 order 信息，并赋值给 `order` 变量。 这用于记录冲突条目的 order，以便后续判断是否需要 split 操作。
*   **`/* entry may have changed before we re-acquire the lock */`**:  注释说明在重新获取锁之前，`entry` 可能已经发生改变。
*   **`if (alloced_order && (old != alloced_shadow || order != alloced_order)) { ... }`**:  条件判断是否需要销毁之前分配的 shadow 项。
    *   `alloced_order`:  检查 `alloced_order` 是否非零，表示之前分配了 shadow 项。
    *   `(old != alloced_shadow || order != alloced_order)`:  检查当前冲突条目 `old` 是否与之前分配的 shadow 项 `alloced_shadow` 不同，或者冲突条目的 order `order` 是否与之前分配的 shadow 项的 order `alloced_order` 不同。 如果不同，则表示之前分配的 shadow 项不再适用，需要销毁。
    *   **`xas_destroy(&xas);`**:  销毁 xarray 状态 `xas` 中可能存在的 shadow 项。
    *   **`alloced_order = 0;`**:  将 `alloced_order` 重置为 0，表示不再有已分配的 shadow 项。
*   **`if (old) { ... }`**:  如果 `old` 指针非空，表示存在冲突条目。
    *   **`if (order > 0 && order > folio_order(folio)) { ... }`**:  条件判断是否需要进行 split 操作。
        *   `order > 0`:  检查冲突条目的 order 是否大于 0。
        *   `order > folio_order(folio)`:  检查冲突条目的 order 是否大于要添加的 folio 的 order。 如果冲突条目的 order 更大，则可能需要进行 split 操作。
        *   **`/* ... 注释 ... */`**:  注释提出了如何处理大型 swap 条目的问题，并断言了在 shmem mapping 中不应该出现这种情况。
        *   **`BUG_ON(shmem_mapping(mapping));`**:  断言检查，在 shmem mapping 中不应该出现需要 split 的情况。
        *   **`if (!alloced_order) { ... }`**:  如果 `alloced_order` 为 0，表示还没有分配 shadow 项。
            *   **`split_order = order;`**:  将冲突条目的 order `order` 赋值给 `split_order` 变量，标记需要进行 split 操作。
            *   **`goto unlock;`**:  跳转到 `unlock` 标签，释放锁并退出循环。  在 `unlock` 标签之后的代码会处理 split 操作。
        *   **`xas_split(&xas, old, order);`**:  执行 xarray split 操作。 将冲突条目 `old` 按照其 order `order` 进行 split。  Split 操作可能用于将一个大的 xarray 条目分割成多个小的条目，以便容纳新的 folio。
        *   **`xas_reset(&xas);`**:  重置 xarray 状态 `xas`。
    *   **`if (shadowp) *shadowp = old;`**:  如果 `shadowp` 指针非空，则将冲突条目 `old` 赋值给 `*shadowp`，通过 `shadowp` 返回 shadow 项。
*   **`xas_store(&xas, folio);`**:  **将 folio 存储到 xarray 中。 这就是将 folio 和 index 关联起来，并放入 page cache 的关键步骤。  `xas_store` 使用 `xas` 状态中记录的 `index` 作为键，将 `folio` 作为值存储到 `mapping->i_pages` xarray 中。**
*   **`if (xas_error(&xas)) goto unlock;`**:  检查 `xas_store` 操作是否发生错误。 如果有错误，则跳转到 `unlock` 标签进行错误处理。
*   **`mapping->nrpages += nr;`**:  更新地址空间 `mapping` 的 `nrpages` 计数器，增加 `nr` (folio 包含的页数)。 记录地址空间中页面的总数。
*   **`/* hugetlb pages do not participate in page cache accounting */`**:  注释说明 hugetlb 页面不参与页缓存的统计。
*   **`if (!huge) { ... }`**:  如果 `folio` 不是 hugetlb 页，则进行页缓存统计更新。
    *   **`__lruvec_stat_mod_folio(folio, NR_FILE_PAGES, nr);`**:  更新 LRU 向量统计信息，增加 `NR_FILE_PAGES` 计数器，增加的数量为 `nr`。 统计文件页面的数量。
    *   **`if (folio_test_pmd_mappable(folio)) __lruvec_stat_mod_folio(folio, NR_FILE_THPS, nr);`**:  如果 folio 是 PMD (Page Middle Directory) 可映射的 (可能表示是 THP - Transparent Huge Page)，则更新 LRU 向量统计信息，增加 `NR_FILE_THPS` 计数器，增加的数量为 `nr`。 统计文件 THP 页面的数量。
*   **`unlock:`**:  标签。
*   **`xas_unlock_irq(&xas);`**:  释放 xarray 状态 `xas` 的自旋锁，并重新启用中断。  **解锁 xarray。**
*   **`/* split needed, alloc here and retry. */`**:  注释说明如果需要 split 操作，则在这里分配 shadow 项并重试。
*   **`if (split_order) { ... }`**:  如果 `split_order` 非零，表示之前标记了需要进行 split 操作。
    *   **`xas_split_alloc(&xas, old, split_order, gfp);`**:  分配用于 split 操作的 shadow 项。
    *   **`if (xas_error(&xas)) goto error;`**:  检查 `xas_split_alloc` 操作是否发生错误。 如果有错误，则跳转到 `error` 标签进行错误处理。
    *   **`alloced_shadow = old;`**:  将分配的 shadow 项 `old` 赋值给 `alloced_shadow` 变量，记录已分配的 shadow 项。
    *   **`alloced_order = split_order;`**:  将 `split_order` 赋值给 `alloced_order` 变量，记录已分配 shadow 项的 order。
    *   **`xas_reset(&xas);`**:  重置 xarray 状态 `xas`。
    *   **`continue;`**:  继续下一次循环迭代，重试添加 folio 的操作。  重试时，由于已经进行了 split 操作，冲突应该已经解决。
*   **`if (!xas_nomem(&xas, gfp)) break;`**:  检查 xarray 状态 `xas` 是否因为内存不足 (`-ENOMEM`) 而出错。
    *   `!xas_nomem(&xas, gfp)`:  如果 `xas_nomem` 返回 false，表示没有内存不足错误，则跳出循环。 如果 `xas_nomem` 返回 true，表示内存不足，则继续循环，等待内存回收。
*   **`if (xas_error(&xas)) goto error;`**:  在循环结束后，再次检查 xarray 状态 `xas` 是否有错误。 如果有错误 (除了 `-ENOMEM` 之外的其他错误)，则跳转到 `error` 标签进行错误处理。
*   **`trace_mm_filemap_add_to_page_cache(folio);`**:  使用 tracing 系统记录 `filemap_add_to_page_cache` 事件，用于性能分析和调试。
*   **`return 0;`**:  如果所有操作都成功完成，则返回 0，表示添加 folio 成功。
*   **`error:`**:  标签，用于错误处理。
*   **`folio->mapping = NULL;`**:  在错误情况下，将 folio 的 `mapping` 成员设置为 `NULL`，解除 folio 与地址空间的关联。
*   **`/* Leave page->index set: truncation relies upon it */`**:  注释说明保留 `folio->index` 的设置，因为 truncation 操作依赖于它。
*   **`folio_put_refs(folio, nr);`**:  减少 folio 的引用计数，减少的数量为 `nr`。 释放之前增加的引用计数。
*   **`return xas_error(&xas);`**:  返回 xarray 状态 `xas` 中记录的错误码。

**总结 `__filemap_add_folio`:**

`__filemap_add_folio` 函数是向页缓存 xarray (`mapping->i_pages`) 添加 folio 的核心函数。 它负责：

1.  **初始化和断言检查**:  初始化 XA_STATE，进行各种断言检查，确保输入参数的有效性。
2.  **准备 Folio**:  设置 folio 的 `mapping` 和 `index` 成员，增加 folio 的引用计数。
3.  **冲突处理循环**:  在一个无限循环中，处理 xarray 中可能存在的冲突。
    *   **查找冲突**:  使用 `xas_for_each_conflict` 查找与要添加的 folio 索引范围冲突的现有条目。
    *   **冲突处理**:  如果找到冲突条目，根据冲突类型和 order 进行处理，可能需要进行 xarray split 操作。
    *   **存储 Folio**:  使用 `xas_store(&xas, folio)` 将 folio 存储到 xarray 中，**以 `index` 作为键，`folio` 作为值。**
4.  **更新统计信息**:  更新地址空间的 `nrpages` 计数器和 LRU 向量统计信息。
5.  **错误处理和重试**:  处理内存不足等错误，并进行重试。
6.  **Tracing**:  记录 tracing 事件。
7.  **返回结果**:  返回操作结果。

**回答你的问题：`filemap_add_folio` 是不是会将 folio 和对应的 index 作为键值对一起放到 page cache 之中啊?**

**是的，你的理解是正确的！ `__filemap_add_folio` 函数中的 `xas_store(&xas, folio);` 这一行代码，正是将 `folio` 和对应的 `index` 作为键值对一起放到 page cache 之中。**

*   **键 (Key):**  `index` (页偏移)  被用作键。  Xarray 的结构本身就是基于索引 (键) 来组织和检索条目的。
*   **值 (Value):**  `folio` 指针被用作值。  存储在 xarray 中的是 `folio` 结构体的指针。

**`mapping->i_pages` xarray 就是页缓存的底层数据结构，它本质上是一个高效的键值存储系统，用于将页索引 (键) 映射到 folio (值)。**  `filemap_add_folio` 和 `__filemap_add_folio` 函数就是用来向这个键值存储系统中添加新的键值对 (index-folio) 的。  后续的页缓存查找操作 (例如 `filemap_get_read_batch`, `find_get_folio`) 就可以使用页索引作为键，在 `mapping->i_pages` xarray 中快速查找和检索到对应的 folio。

