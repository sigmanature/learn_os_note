## `__filemap_get_folio(struct address_space *mapping, pgoff_t index, fgf_t fgp_flags, gfp_t gfp)`
* **重要函数:** [`filemap_get_entry`](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/filemap.c/filemap_get_entry.md)
```c
struct folio *__filemap_get_folio(struct address_space *mapping, pgoff_t index,
		fgf_t fgp_flags, gfp_t gfp)
{
	struct folio *folio;

repeat:
	folio = filemap_get_entry(mapping, index);
	if (xa_is_value(folio))
		folio = NULL;
	if (!folio)
		goto no_page;

	if (fgp_flags & FGP_LOCK) {
		if (fgp_flags & FGP_NOWAIT) {
			if (!folio_trylock(folio)) {
				folio_put(folio);
				return ERR_PTR(-EAGAIN);
			}
		} else {
			folio_lock(folio);
		}

		/* Has the page been truncated? */
		if (unlikely(folio->mapping != mapping)) {
			folio_unlock(folio);
			folio_put(folio);
			goto repeat;
		}
		VM_BUG_ON_FOLIO(!folio_contains(folio, index), folio);
	}

	if (fgp_flags & FGP_ACCESSED)
		folio_mark_accessed(folio);
	else if (fgp_flags & FGP_WRITE) {
		/* Clear idle flag for buffer write */
		if (folio_test_idle(folio))
			folio_clear_idle(folio);
	}

	if (fgp_flags & FGP_STABLE)
		folio_wait_stable(folio);
no_page:
	if (!folio && (fgp_flags & FGP_CREAT)) {
		unsigned int min_order = mapping_min_folio_order(mapping);
		unsigned int order = max(min_order, FGF_GET_ORDER(fgp_flags));
		int err;
		index = mapping_align_index(mapping, index);

		if ((fgp_flags & FGP_WRITE) && mapping_can_writeback(mapping))
			gfp |= __GFP_WRITE;
		if (fgp_flags & FGP_NOFS)
			gfp &= ~__GFP_FS;
		if (fgp_flags & FGP_NOWAIT) {
			gfp &= ~GFP_KERNEL;
			gfp |= GFP_NOWAIT | __GFP_NOWARN;
		}
		if (WARN_ON_ONCE(!(fgp_flags & (FGP_LOCK | FGP_FOR_MMAP))))
			fgp_flags |= FGP_LOCK;

		if (order > mapping_max_folio_order(mapping))
			order = mapping_max_folio_order(mapping);
		/* If we're not aligned, allocate a smaller folio */
		if (index & ((1UL << order) - 1))
			order = __ffs(index);

		do {
			gfp_t alloc_gfp = gfp;

			err = -ENOMEM;
			if (order > min_order)
				alloc_gfp |= __GFP_NORETRY | __GFP_NOWARN;
			folio = filemap_alloc_folio(alloc_gfp, order);
			if (!folio)
				continue;

			/* Init accessed so avoid atomic mark_page_accessed later */
			if (fgp_flags & FGP_ACCESSED)
				__folio_set_referenced(folio);

			err = filemap_add_folio(mapping, folio, index, gfp);
			if (!err)
				break;
			folio_put(folio);
			folio = NULL;
		} while (order-- > min_order);

		if (err == -EEXIST)
			goto repeat;
		if (err)
			return ERR_PTR(err);
		/*
		 * filemap_add_folio locks the page, and for mmap
		 * we expect an unlocked page.
		 */
		if (folio && (fgp_flags & FGP_FOR_MMAP))
			folio_unlock(folio);
	}

	if (!folio)
		return ERR_PTR(-ENOENT);
	return folio;
}
EXPORT_SYMBOL(__filemap_get_folio);
```

*   **函数签名：**
    ```c
    struct folio *__filemap_get_folio(struct address_space *mapping, pgoff_t index,
        fgf_t fgp_flags, gfp_t gfp)
    ```
    *   返回 `struct folio *`。这是从页缓存获取 folio 的核心函数。

*   **缓存查找：**
    ```c
	repeat:
	folio = filemap_get_entry(mapping, index);
	if (xa_is_value(folio))
		folio = NULL;
	if (!folio)
		goto no_page;
    ```
    *   `repeat:`: 用于潜在重试的标签。
    *   `folio = filemap_get_entry(mapping, index);`: 这是在页缓存中查找 folio 的主要函数。
        *   `filemap_get_entry(mapping, index)`: 在与给定 `index` 关联的 `address_space` 的基数树（或 xarray，取决于内核版本）中查找 folio。如果找到，则返回 `struct folio *`，如果未找到或存在特殊条目（如影子条目），则返回特殊值。
    *   `if (xa_is_value(folio)) folio = NULL;`: `xa_is_value(folio)` 检查返回值是否是特殊值（如 `XA_RETRY` 或 `XA_ZERO_ENTRY`）而不是有效的 folio 指针。如果是，则将 `folio` 设置为 `NULL`，以指示未找到有效的 folio。
    *   `if (!folio) goto no_page;`: 如果在查找后 `folio` 为 `NULL`，则表示在缓存中未找到 folio，因此跳转到 `no_page` 标签。

*   **处理现有 Folio（如果在缓存中找到）：**
    ```c
	if (fgp_flags & FGP_LOCK) {
		if (fgp_flags & FGP_NOWAIT) {
			if (!folio_trylock(folio)) {
				folio_put(folio);
				return ERR_PTR(-EAGAIN);
			}
		} else {
			folio_lock(folio);
		}

		/* Has the page been truncated? */
		if (unlikely(folio->mapping != mapping)) {
			folio_unlock(folio);
			folio_put(folio);
			goto repeat;
		}
		VM_BUG_ON_FOLIO(!folio_contains(folio, index), folio);
	}

	if (fgp_flags & FGP_ACCESSED)
		folio_mark_accessed(folio);
	else if (fgp_flags & FGP_WRITE) {
		/* Clear idle flag for buffer write */
		if (folio_test_idle(folio))
			folio_clear_idle(folio);
	}

	if (fgp_flags & FGP_STABLE)
		folio_wait_stable(folio);
    ```
    *   **`FGP_LOCK` 标志：** 如果 `fgp_flags` 中设置了 `FGP_LOCK`，则表示调用者想要锁定 folio。
        *   **`FGP_NOWAIT` 标志（在 `FGP_LOCK` 内）：** 如果也设置了 `FGP_NOWAIT`，它会尝试使用 `folio_trylock(folio)` 以非阻塞方式获取锁。如果无法立即获取锁，它将释放 folio (`folio_put(folio)`) 并返回 `-EAGAIN` 错误。
        *   **阻塞锁（在 `FGP_LOCK` 内）：** 如果未设置 `FGP_NOWAIT`，它将使用 `folio_lock(folio)` 以阻塞方式获取锁。
        *   **截断检查：** 获取锁后，它会检查 folio 的 `mapping` 是否仍然是预期的 `mapping`。如果不是，则表示 folio 可能已被截断或重新映射，因此它会释放锁和 folio 并从头开始重试 (`goto repeat`)。
        *   `VM_BUG_ON_FOLIO(!folio_contains(folio, index), folio);`: 一个健全性检查，以确保 folio 实际上包含请求的 `index`。

    *   **`FGP_ACCESSED` 标志：** 如果设置了 `FGP_ACCESSED`，它会使用 `folio_mark_accessed(folio)` 将 folio 标记为已访问。这将更新 folio 的 LRU（最近最少使用）状态，使其保留在活动 LRU 列表中。

    *   **`FGP_WRITE` 标志：** 如果设置了 `FGP_WRITE`，并且如果 folio 当前被标记为空闲 (`folio_test_idle(folio)`)，它将使用 `folio_clear_idle(folio)` 清除空闲标志。这可能与回写和缓冲区管理有关，表明即将在此 folio 上发生写入操作。

    *   **`FGP_STABLE` 标志：** 如果设置了 `FGP_STABLE`，它将使用 `folio_wait_stable(folio)` 等待 folio 变为稳定状态。稳定的 folio 意味着对 folio 的任何正在进行的写入操作都已完成，并且数据在磁盘上（或至少在回写缓存中）是一致的。

*   **处理缓存未命中 (`no_page` 标签)：**
  	```c
	no_page:
	if (!folio && (fgp_flags & FGP_CREAT)) {
        ... // Folio 创建逻辑
    }

	if (!folio)
		return ERR_PTR(-ENOENT);
	return folio;
	```
    *   `no_page:`: 当在缓存中未找到 folio 时到达的标签 (`!folio`)。
    *   **`FGP_CREAT` 标志：** 如果在 `fgp_flags` 中设置了 `FGP_CREAT`，则表示调用者希望在缓存中未找到 folio 时创建一个新的 folio。
        *   **Folio 阶数 (Order) 计算:**
    ```c
    unsigned int min_order = mapping_min_folio_order(mapping);
    unsigned int order = max(min_order, FGF_GET_ORDER(fgp_flags));
    int err;
    index = mapping_align_index(mapping, index);

    if (order > mapping_max_folio_order(mapping))
        order = mapping_max_folio_order(mapping);
    /* If we're not aligned, allocate a smaller folio */
    if (index & ((1UL << order) - 1))
        order = __ffs(index);
    ```
    *   **`unsigned int min_order = mapping_min_folio_order(mapping);`:**  获取与 `mapping` 关联的最小 folio 阶数。不同的 `address_space` 可能有不同的最小阶数要求。
    *   **`unsigned int order = max(min_order, FGF_GET_ORDER(fgp_flags));`:**  计算初始的 folio 阶数。
        *   `FGF_GET_ORDER(fgp_flags)`:  尝试从 `fgp_flags` 中获取请求的 folio 阶数。`FGF_GET_ORDER` 可能是个宏，用于从标志中提取阶数信息。如果 `fgp_flags` 中没有指定阶数，则 `FGF_GET_ORDER` 可能会返回 0。
        *   `max(min_order, FGF_GET_ORDER(fgp_flags))`:  取 `min_order` 和从 `fgp_flags` 获取的阶数中的较大值。**这意味着，即使 `fgp_flags` 没有指定阶数，也会至少分配 `min_order` 阶的 folio。**
    *   **`index = mapping_align_index(mapping, index);`:**  将 `index` 对齐到 folio 大小的边界。对于多页 folio，索引通常需要对齐到 folio 的起始位置。
    *   **`if (order > mapping_max_folio_order(mapping)) order = mapping_max_folio_order(mapping);`:**  限制 folio 阶数不超过与 `mapping` 关联的最大允许阶数。
    *   **不对齐索引的特殊处理:**
        ```c
        /* If we're not aligned, allocate a smaller folio */
        if (index & ((1UL << order) - 1))
            order = __ffs(index);
        ```
        *   `index & ((1UL << order) - 1)`:  检查 `index` 是否在当前 `order` 下是对齐的。如果 `index` 的低 `order` 位不为 0，则表示不对齐。
        *   `order = __ffs(index);`:  如果索引不对齐，则 **强制将 `order` 降低到足以包含 `index` 的最小阶数**。`__ffs(index)` 函数返回 `index` 二进制表示中第一个 set bit 的位置，这可以用来计算最小的阶数。**这个逻辑是为了处理一些特殊情况，可能需要分配一个比通常小的 folio。**

*   **Folio 分配循环 (`do-while` 循环):**
    ```c
    do {
        gfp_t alloc_gfp = gfp;

        err = -ENOMEM;
        if (order > min_order)
            alloc_gfp |= __GFP_NORETRY | __GFP_NOWARN;
        folio = filemap_alloc_folio(alloc_gfp, order);
        if (!folio)
            continue;

        /* Init accessed so avoid atomic mark_page_accessed later */
        if (fgp_flags & FGP_ACCESSED)
            __folio_set_referenced(folio);

        err = filemap_add_folio(mapping, folio, index, gfp);
        if (!err)
            break;
        folio_put(folio);
        folio = NULL;
    } while (order-- > min_order);
    ```
    *   **`do-while (order-- > min_order)` 循环:**  这是一个 **阶数递减的分配重试循环**。它从计算出的 `order` 开始，尝试分配 folio，如果分配失败，则 **逐步降低 `order` (每次减 1)**，并再次尝试分配，直到 `order` 降低到 `min_order` 为止。
    *   **`gfp_t alloc_gfp = gfp;`:**  复制传入的 `gfp` 标志到 `alloc_gfp`。
    *   **`if (order > min_order) alloc_gfp |= __GFP_NORETRY | __GFP_NOWARN;`:**  如果当前尝试的 `order` 大于最小阶数 `min_order`，则在 `alloc_gfp` 中 **添加 `__GFP_NORETRY` 和 `__GFP_NOWARN` 标志**。
        *   `__GFP_NORETRY`:  内存分配失败时 **不重试**。
        *   `__GFP_NOWARN`:  内存分配失败时 **不打印警告信息**。
        *   **逻辑:**  对于较大阶数的 folio 分配，如果第一次尝试失败，则不再重试，也不打印警告，直接尝试分配更小阶数的 folio。这可能是为了避免在内存压力较大时，长时间阻塞在较大 folio 的分配上。
    *   **`folio = filemap_alloc_folio(alloc_gfp, order);`:**  调用 `filemap_alloc_folio` 函数 **分配指定 `order` 的 folio**。
    *   **`if (!folio) continue;`:**  如果 `filemap_alloc_folio` 返回 `NULL` (分配失败)，则 `continue` 到循环的下一次迭代，尝试分配更小阶数的 folio。
    *   **`if (fgp_flags & FGP_ACCESSED) __folio_set_referenced(folio);`:**  如果 `fgp_flags` 中设置了 `FGP_ACCESSED` 标志，则调用 `__folio_set_referenced(folio)` 将新分配的 folio 标记为已引用。
    *   **`err = filemap_add_folio(mapping, folio, index, gfp);`:**  调用 `filemap_add_folio` 函数将新分配的 `folio` **添加到与 `mapping` 关联的页面缓存 XArray 中**，索引为 `index`。
    *   **`if (!err) break;`:**  如果 `filemap_add_folio` 返回 0 (添加成功)，则 `break` 跳出循环，表示 folio 分配和添加成功。
    *   **`folio_put(folio); folio = NULL;`:**  如果 `filemap_add_folio` 返回错误 (添加失败)，则 **释放已分配的 `folio`**，并将 `folio` 设置为 `NULL`，以便在下一次循环迭代中重新分配。
    *   **`-EEXIST` 处理:**
        ```c
        if (err == -EEXIST)
            goto repeat;
        ```
        *   如果 `filemap_add_folio` 返回 `-EEXIST` 错误，表示 **在添加 folio 的过程中，发现页面缓存中已经存在相同索引的 folio**。这可能是并发操作导致的。在这种情况下，`goto repeat` 跳转回 `repeat` 标签， **重新开始整个页面获取流程**，包括重新查找页面缓存。
    *   **其他错误处理:**
        ```c
        if (err)
            return ERR_PTR(err);
        ```
        *   如果 `filemap_add_folio` 返回其他错误 (非 `-EEXIST`)，则表示 folio 分配或添加过程中发生了不可恢复的错误，函数返回错误指针 `ERR_PTR(err)`。

*   **`FGP_FOR_MMAP` 特殊处理:**
    ```c
    /*
     * filemap_add_folio locks the page, and for mmap
     * we expect an unlocked page.
     */
    if (folio && (fgp_flags & FGP_FOR_MMAP))
        folio_unlock(folio);
    ```
    *   如果 `fgp_flags` 中设置了 `FGP_FOR_MMAP` 标志 (表示用于内存映射 mmap)，则在成功分配和添加 folio 后，需要 **手动解锁 folio**。这是因为 mmap 场景下，通常期望获取到的是未锁定的页面。


    *   **没有 Folio 且没有 `FGP_CREAT`：** 如果在 `no_page` 标签之后 `folio` 仍然为 `NULL` 并且未设置 `FGP_CREAT`，则表示未找到 folio 并且未请求创建。在这种情况下，它使用 `ERR_PTR(-ENOENT)` 返回 `-ENOENT` 错误。

    *   **成功返回：** 如果在缓存中找到 folio 或创建了新的 folio（并且没有发生错误），则该函数返回 `struct folio *`。

*   **`__filemap_get_folio` 的目的：**
    *   **核心页缓存检索：** 这是从页缓存获取 folio 的中心函数。
    *   **缓存查找和创建：** 它处理查找现有 folio 和在请求时创建新 folio (`FGP_CREAT`)。
    *   **锁定和标志：** 它遵循各种标志 (`FGP_LOCK`、`FGP_NOWAIT`、`FGP_ACCESSED`、`FGP_WRITE`、`FGP_STABLE`、`FGP_CREAT`) 来控制 folio 检索和锁定行为。
    *   **错误处理：** 它处理各种错误情况，包括缓存查找失败、分配失败和锁定失败。
    *   **Folio 管理：** 它使用 `struct folio` 作为页缓存管理的主要单元。
