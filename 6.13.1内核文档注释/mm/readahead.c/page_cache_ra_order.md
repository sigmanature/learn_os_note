 好的，让我们来深入解析 `page_cache_ra_order` 函数，并解答你的疑问。

**1. `page_cache_ra_order` 函数代码解析**

```c
void page_cache_ra_order(struct readahead_control *ractl,
		struct file_ra_state *ra, unsigned int new_order)
{
	struct address_space *mapping = ractl->mapping;
	pgoff_t index = readahead_index(ractl);/*已经被设置为ra中的start了*/
	unsigned int min_order = mapping_min_folio_order(mapping);
	pgoff_t limit = (i_size_read(mapping->host) - 1) >> PAGE_SHIFT;
	pgoff_t mark = index + ra->size - ra->async_size;
	unsigned int nofs;
	int err = 0;
	gfp_t gfp = readahead_gfp_mask(mapping);
	unsigned int min_ra_size = max(4, mapping_min_folio_nrpages(mapping));
```

* **`void page_cache_ra_order(struct readahead_control *ractl, struct file_ra_state *ra, unsigned int new_order)`**: 函数定义。
    * `void`: 无返回值。
    * 参数：
        * `struct readahead_control *ractl`: 预读控制结构体。
        * `struct file_ra_state *ra`: 文件预读状态。
        * `unsigned int new_order`:  建议的 folio order (以 2 为底的对数)。
* **`struct address_space *mapping = ractl->mapping;`**: 获取地址空间 `mapping`。
* **`pgoff_t index = readahead_index(ractl);`**: 获取预读起始页索引 `index`。注释说明 `index` 已经被设置为 `ra->start` (在 `page_cache_sync_ra` 中)。
* **`unsigned int min_order = mapping_min_folio_order(mapping);`**: 获取地址空间支持的最小 folio order `min_order`。
* **`pgoff_t limit = (i_size_read(mapping->host) - 1) >> PAGE_SHIFT;`**: 计算文件结束页索引 `limit`。
* **`pgoff_t mark = index + ra->size - ra->async_size;`**: 计算 `mark` 值。与 `page_cache_ra_unbounded` 中的 `mark` 不同，这里的 `mark` 用于限制循环的范围，而不是标记 readahead folio。  它表示预读范围的截止页索引，考虑了 `ra->size` (预读大小) 和 `ra->async_size` (异步预读大小)。
* **`unsigned int nofs;`**: 声明变量 `nofs`，用于保存 `NOFS` 状态。
* **`int err = 0;`**: 初始化错误码 `err` 为 0.
* **`gfp_t gfp = readahead_gfp_mask(mapping);`**: 获取内存分配标志 `gfp`。
* **`unsigned int min_ra_size = max(4, mapping_min_folio_nrpages(mapping));`**: 计算最小预读大小 `min_ra_size`。取 4 和最小 folio 页数 `mapping_min_folio_nrpages(mapping)` 的最大值。

```c
	/*
	 * Fallback when size < min_nrpages as each folio should be
	 * at least min_nrpages anyway.
	 */
	if (!mapping_large_folio_support(mapping) || ra->size < min_ra_size)
		goto fallback;
```

* **注释块**: 解释了当 `ra->size < min_nrpages` 时会 fallback，因为每个 folio 至少应该是 `min_nrpages` 大小。
* **`if (!mapping_large_folio_support(mapping) || ra->size < min_ra_size)`**: **Fallback 条件判断。**
    * `!mapping_large_folio_support(mapping)`: 如果文件系统不支持大 folio (large folio)。
    * `ra->size < min_ra_size`: 如果最近一次预读的大小 `ra->size` 小于最小预读大小 `min_ra_size`。
    * 如果以上任一条件为真，则跳转到 `fallback` 标签。

```c
	limit = min(limit, index + ra->size - 1);

	if (new_order < mapping_max_folio_order(mapping))
		new_order += 2;

	new_order = min(mapping_max_folio_order(mapping), new_order);
	new_order = min_t(unsigned int, new_order, ilog2(ra->size));
	new_order = max(new_order, min_order);
```

* **`limit = min(limit, index + ra->size - 1);`**: 再次限制预读结束页索引 `limit`，取文件结束页索引和预读范围结束页索引的最小值。
* **`if (new_order < mapping_max_folio_order(mapping)) new_order += 2;`**: 如果建议的 `new_order` 小于最大 folio order，则增加 `new_order`，这里增加了 2。 可能是为了尝试使用更大的 folio size。
* **`new_order = min(mapping_max_folio_order(mapping), new_order);`**: 限制 `new_order` 不超过最大 folio order。
* **`new_order = min_t(unsigned int, new_order, ilog2(ra->size));`**: 进一步限制 `new_order`，取 `new_order` 和 $log_2(ra->size)$ 的最小值。 预读大小越大，允许的 folio order 越大。
* **`new_order = max(new_order, min_order);`**: 确保 `new_order` 不小于最小 folio order。 最终确定了本次预读操作使用的 folio order `new_order`。

```c
	/* See comment in page_cache_ra_unbounded() */
	nofs = memalloc_nofs_save();
	filemap_invalidate_lock_shared(mapping);
	/*
	 * If the new_order is greater than min_order and index is
	 * already aligned to new_order, then this will be noop as index
	 * aligned to new_order should also be aligned to min_order.
	 */
	ractl->_index = mapping_align_index(mapping, index);
	index = readahead_index(ractl);
```

* **注释块**:  引用 `page_cache_ra_unbounded()` 中的注释，关于 `NOFS` 和死锁避免。
* **`nofs = memalloc_nofs_save();`**: 保存 `NOFS` 状态并进入 `NOFS` 模式。
* **`filemap_invalidate_lock_shared(mapping);`**: 获取共享失效锁。
* **注释块**: 解释了如果 `new_order` 大于 `min_order` 并且 `index` 已经对齐到 `new_order`，则 `mapping_align_index` 操作是 noop (无操作)，因为对齐到更大的 order 也必然对齐到更小的 order。
* **`ractl->_index = mapping_align_index(mapping, index);`**: 对齐预读起始页索引 `index` 到 `min_order` 的边界。  **注意这里是对齐到 `min_order`，而不是 `new_order`。**  这可能是一个微妙的地方，需要仔细考虑。
* **`index = readahead_index(ractl);`**: 重新获取对齐后的预读起始页索引 `index`。

```c
	while (index <= limit) {
		unsigned int order = new_order;

		/* Align with smaller pages if needed */
		if (index & ((1UL << order) - 1))
			order = __ffs(index);
		/* Don't allocate pages past EOF */
		while (order > min_order && index + (1UL << order) - 1 > limit)
			order--;
		err = ra_alloc_folio(ractl, index, mark, order, gfp);
		if (err)
			break;
		index += 1UL << order;
	}
```

* **`while (index <= limit)`**: 循环，直到预读起始页索引 `index` 超过 `limit` (文件结束页索引或预读范围截止页索引)。
    * **`unsigned int order = new_order;`**:  初始化当前迭代的 folio order `order` 为之前计算的 `new_order`。
    * **`if (index & ((1UL << order) - 1)) order = __ffs(index);`**: **Order 调整逻辑：如果 `index` 没有对齐到当前 `order` 的边界，则减小 `order`。**
        * `(1UL << order) - 1`:  生成一个掩码，用于检查 `index` 是否对齐到 `order` 的边界。
        * `index & ((1UL << order) - 1)`:  如果结果非零，表示 `index` 没有对齐到 `order` 的边界。
        * `order = __ffs(index);`:  `__ffs(index)` (Find First Set bit) 函数返回 `index` 最低设置位的位置。 这实际上计算了 **需要使用的最小 order，使得 `index` 对齐到这个 order 的边界**。  例如，如果 `index = 5` (二进制 `101`)，`__ffs(index)` 返回 0，order 变为 0 (PAGE_SIZE)。 如果 `index = 4` (二进制 `100`)，`__ffs(index)` 返回 2，order 变为 2 (4 * PAGE_SIZE)。
    * **`while (order > min_order && index + (1UL << order) - 1 > limit) order--;`**: **Order 缩小逻辑：如果当前 `order` 大于最小 order 并且使用当前 `order` 会超出文件末尾 `limit`，则减小 `order`。**  循环减小 `order`，直到 `order` 等于 `min_order` 或者使用当前 `order` 不会超出文件末尾。
    * **`err = ra_alloc_folio(ractl, index, mark, order, gfp);`**: **分配并添加 folio。**  `ra_alloc_folio` 函数 (未提供代码)  很可能负责分配指定 `order` 大小的 folio，并将其添加到地址空间，并可能进行一些预读相关的设置。 参数 `mark` 在这里的作用可能与限制预读范围有关。
    * **`if (err) break;`**: 如果 `ra_alloc_folio` 返回错误，则跳出循环。
    * **`index += 1UL << order;`**:  更新预读起始页索引 `index`，增加 $2^{order}$，即跳过已分配的 folio 区域。

```c
	read_pages(ractl);
	filemap_invalidate_unlock_shared(mapping);
	memalloc_nofs_restore(nofs);

	/*
	 * If there were already pages in the page cache, then we may have
	 * left some gaps.  Let the regular readahead code take care of this
	 * situation.
	 */
	if (!err)
		return;
fallback:
	do_page_cache_ra(ractl, ra->size, ra->async_size);
}
```

* **`read_pages(ractl);`**: 提交 I/O 请求。
* **`filemap_invalidate_unlock_shared(mapping);`**: 释放共享失效锁。
* **`memalloc_nofs_restore(nofs);`**: 恢复 `NOFS` 状态。
* **注释块**: 解释了如果页缓存中已经存在页面，可能会留下一些空隙，让常规预读代码处理这种情况。
* **`if (!err) return;`**: 如果没有错误 (`err == 0`)，则函数正常返回。
* **`fallback:`**: **Fallback 标签。**
* **`do_page_cache_ra(ractl, ra->size, ra->async_size);`**: **Fallback 逻辑：调用 `do_page_cache_ra` 函数进行预读。**  使用原始的预读大小 `ra->size` 和异步预读大小 `ra->async_size`。

**2. `page_cache_ra_order` 的逻辑总结**

`page_cache_ra_order` 函数旨在实现一种 **基于 folio order 的预读策略**，它尝试使用 **更大尺寸的 folio** 来进行预读，以提高效率，尤其是在文件系统支持大 folio 的情况下。

**主要逻辑步骤:**

1. **计算和调整 folio order**:  根据建议的 `new_order`、最大/最小 folio order、以及预读大小 `ra->size`，计算出一个合适的 folio order `new_order`。
2. **循环分配 folio**:  在一个循环中，根据计算出的 `new_ord