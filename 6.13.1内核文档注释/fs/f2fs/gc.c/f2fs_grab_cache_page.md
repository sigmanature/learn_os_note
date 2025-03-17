**核心函数**[__filemap_get_folio](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/filemap.c/__filemap_get_folio.md)
  好的，我们来深入、详细地用中文重新解析 `__filemap_get_folio` 函数，并重点关注 folio 的分配条件和阶数计算，同时理清 `f2fs_grab_cache_page` 及其调用链的关系。

**1. `f2fs_grab_cache_page(struct address_space *mapping, pgoff_t index, bool for_write)`**

```c
static inline struct page *f2fs_grab_cache_page(struct address_space *mapping,
						pgoff_t index, bool for_write)
{
	struct page *page;
	unsigned int flags;

	if (IS_ENABLED(CONFIG_F2FS_FAULT_INJECTION)) {
		if (!for_write)
			page = find_get_page_flags(mapping, index,
							FGP_LOCK | FGP_ACCESSED);
		else
			page = find_lock_page(mapping, index);
		if (page)
			return page;

		if (time_to_inject(F2FS_M_SB(mapping), FAULT_PAGE_ALLOC))
			return NULL;
	}

	if (!for_write)
		return grab_cache_page(mapping, index);

	flags = memalloc_nofs_save();
	page = grab_cache_page_write_begin(mapping, index);
	memalloc_nofs_restore(flags);

	return page;
}
```

*   **功能概述:** `f2fs_grab_cache_page` 是 F2FS 特有的函数，用于从页面缓存中获取页面。它根据 `for_write` 参数区分读操作和写操作，并加入了 F2FS 的故障注入机制。

*   **故障注入 (CONFIG_F2FS_FAULT_INJECTION):**
    ```c
    if (IS_ENABLED(CONFIG_F2FS_FAULT_INJECTION)) {
        if (!for_write)
            page = find_get_page_flags(mapping, index,
                            FGP_LOCK | FGP_ACCESSED);
        else
            page = find_lock_page(mapping, index);
        if (page)
            return page;

        if (time_to_inject(F2FS_M_SB(mapping), FAULT_PAGE_ALLOC))
            return NULL;
    }
    ```
    *   `IS_ENABLED(CONFIG_F2FS_FAULT_INJECTION)`:  检查是否启用了 F2FS 故障注入配置。
    *   **读操作 (`!for_write`):**
        *   `page = find_get_page_flags(mapping, index, FGP_LOCK | FGP_ACCESSED);`:  尝试使用 `find_get_page_flags` 函数 **非创建地** 查找页面。`FGP_LOCK | FGP_ACCESSED` 标志表示如果找到页面，则对其加锁并标记为已访问。
    *   **写操作 (`for_write`):**
        *   `page = find_lock_page(mapping, index);`:  尝试使用 `find_lock_page` 函数 **非创建地** 查找页面并加锁。
    *   `if (page) return page;`:  如果 `find_get_page_flags` 或 `find_lock_page` 找到了页面，则直接返回，不再进行后续的创建操作。
    *   `if (time_to_inject(F2FS_M_SB(mapping), FAULT_PAGE_ALLOC)) return NULL;`:  如果页面未找到，并且需要注入 `FAULT_PAGE_ALLOC` 类型的故障，则模拟页面分配失败，返回 `NULL`。

*   **正常页面获取 (无故障注入或故障未触发):**
    ```c
    if (!for_write)
        return grab_cache_page(mapping, index);

    flags = memalloc_nofs_save();
    page = grab_cache_page_write_begin(mapping, index);
    memalloc_nofs_restore(flags);

    return page;
    ```
    *   **读操作 (`!for_write`):**
        *   `return grab_cache_page(mapping, index);`:  直接调用 `grab_cache_page` 函数来获取页面。
    *   **写操作 (`for_write`):**
        *   `flags = memalloc_nofs_save();`:  保存 `__GFP_NOFS` 标志的当前状态。`__GFP_NOFS` 标志用于限制内存分配操作是否可以进行文件系统相关的操作（例如，是否可以触发文件系统的回写）。
        *   `page = grab_cache_page_write_begin(mapping, index);`:  调用 `grab_cache_page_write_begin` 函数来获取页面，这个函数会为写操作做一些特殊的准备工作，例如可能涉及到日志记录或预留空间。
        *   `memalloc_nofs_restore(flags);`:  恢复 `__GFP_NOFS` 标志到之前的状态。

*   **总结 `f2fs_grab_cache_page`:**  `f2fs_grab_cache_page` 是 F2FS 获取页面缓存页面的主要入口。它根据读写操作类型选择不同的获取函数，并集成了故障注入机制用于测试。在 `f2fs_get_sum_page` 的上下文中，由于是读取 Summary Page，`for_write` 参数为 `false`，因此会走 `grab_cache_page(mapping, index)` 的路径。

**2. `grab_cache_page(struct address_space *mapping, pgoff_t index)` 和 `find_or_create_page(struct address_space *mapping, pgoff_t index, gfp_t gfp_mask)`**

```c
static inline struct page *grab_cache_page(struct address_space *mapping,
								pgoff_t index)
{
	return find_or_create_page(mapping, index, mapping_gfp_mask(mapping));
}

static inline struct page *find_or_create_page(struct address_space *mapping,
					pgoff_t index, gfp_t gfp_mask)
{
	return pagecache_get_page(mapping, index,
					FGP_LOCK|FGP_ACCESSED|FGP_CREAT,
					gfp_mask);
}
```

*   **`grab_cache_page`:**  `grab_cache_page` 函数非常简单，它直接调用 `find_or_create_page`，并将 `mapping_gfp_mask(mapping)` 作为 `gfp_mask` 参数传递。`mapping_gfp_mask(mapping)` 用于获取与 `address_space` 关联的默认内存分配标志。

*   **`find_or_create_page`:**  `find_or_create_page` 函数也比较简单，它直接调用 `pagecache_get_page`，并传递了以下标志：
    *   `FGP_LOCK`:  如果找到或创建了页面，则对其加锁。
    *   `FGP_ACCESSED`:  如果找到或创建了页面，则标记为已访问。
    *   `FGP_CREAT`:  **关键标志！**  如果页面在页面缓存中未找到，则 **创建** 一个新的页面。
    *   `gfp_mask`:  内存分配标志，从 `grab_cache_page` 传递下来，通常是 `mapping_gfp_mask(mapping)`。

*   **总结 `grab_cache_page` 和 `find_or_create_page`:**  这两个函数都是对 `pagecache_get_page` 的简单封装，主要目的是 **以 "查找或创建" 的方式获取页面缓存页**。 `find_or_create_page` 通过 `FGP_CREAT` 标志控制了在页面未找到时创建新页面的行为，并默认加上了 `FGP_LOCK` 和 `FGP_ACCESSED` 标志。 `grab_cache_page` 进一步简化了调用，使用了 `address_space` 默认的内存分配标志。