```c
/*
 * 将 folio 标记为脏，并在页缓存中将其设置为脏。
 *
 * 如果 warn 为 true，则当 folio 未更新且未被截断时，发出警告。
 *
 * 调用者有责任防止在此函数进行期间 folio 被截断，尽管它可能在此函数调用之前已被截断。
 * 大多数调用者都锁定了 folio。
 * 少数人通过其他方式阻止 folio 被截断（例如，zap_vma_pages() 已映射它并持有页表锁）。
 * 当从 mark_buffer_dirty() 调用时，文件系统应持有对正在标记为脏的 buffer_head 的引用，这会导致
 * try_to_free_buffers() 失败。
 */
void __folio_mark_dirty(struct folio *folio, struct address_space *mapping,
			     int warn)
{
	unsigned long flags;

	xa_lock_irqsave(&mapping->i_pages, flags);
	if (folio->mapping) {	/* 与截断竞争？ */
		WARN_ON_ONCE(warn && !folio_test_uptodate(folio));
		folio_account_dirtied(folio, mapping);
		__xa_set_mark(&mapping->i_pages, folio_index(folio),
				PAGECACHE_TAG_DIRTY);
	}
	xa_unlock_irqrestore(&mapping->i_pages, flags);
}
```

* **目的:** 实际在页缓存的 XArray 中标记 folio 为脏并执行记账的底层函数。
* **参数:**
    * `folio`: `folio`。
    * `mapping`: `address_space`。
    * `warn`: 控制是否在 folio 未更新时发出警告的标志。
* **功能:**
    1. **获取 XArray 锁:** `xa_lock_irqsave(&mapping->i_pages, flags);` 这获取保护 `address_space` 的 `i_pages` XArray 的自旋锁。此锁对于在修改页缓存的数据结构时进行并发控制至关重要。 `_irqsave` 在持有锁时本地禁用中断。
    2. **检查 `folio->mapping`:** `if (folio->mapping) { ... }` 它检查 `folio->mapping` 是否仍然有效。这是针对与 `truncate` 竞争的检查。如果 `truncate` 已经从映射中删除了 folio，则 `folio->mapping` 将为 `NULL`。
    3. **更新警告:** `WARN_ON_ONCE(warn && !folio_test_uptodate(folio));` 如果 `warn` 标志为 true 并且 folio 未标记为更新，则发出警告（每个 folio 一次）。这是由 `filemap_dirty_folio` 中的 `!folio_test_private(folio)` 参数控制的警告。这是一个健全性检查，用于捕获 folio 正在变脏但未被认为与磁盘状态一致的情况。
    4. **记账脏页:** `folio_account_dirtied(folio, mapping);` 调用 `folio_account_dirtied` 以更新各种统计信息和写回相关的计数器。
    5. **在 XArray 中设置脏标记:** `__xa_set_mark(&mapping->i_pages, folio_index(folio), PAGECACHE_TAG_DIRTY);` 这是核心的页缓存操作。它在 `i_pages` XArray 中为 folio 设置 `PAGECACHE_TAG_DIRTY` 标记。 XArray 由 folio 索引索引，此标记由写回机制用于识别需要写回磁盘的脏 folio。
    6. **释放 XArray 锁:** `xa_unlock_irqrestore(&mapping->i_pages, flags);` 释放 XArray 自旋锁，重新启用中断。

* **并发与截断安全:**
    * **XArray 自旋锁:** `xa_lock_irqsave` 和 `xa_unlock_irqrestore` 调用提供主要的并发控制。自旋锁保护 `i_pages` XArray 免受并发修改，确保脏页标记、添加/删除 folio 和写回等操作被序列化和一致。
    * **`folio->mapping` 检查:** `if (folio->mapping)` 检查是针对 `truncate` 的关键竞争检测机制。如果在锁定部分内 `folio->mapping` 为 `NULL`，则意味着 `truncate` 已经删除了 folio，并且该函数避免对可能已释放或无效的映射进行操作。
    * **调用者责任（再次）:** 注释 “调用者有责任防止在此函数进行期间 folio 被截断...” 重申，虽然 `__folio_mark_dirty` 提供了一些内部检查和锁定，但更高级别的调用者（如 `filemap_dirty_folio`、`f2fs_dirty_data_folio`、`folio_mark_dirty`）仍然负责确保在整个脏页标记过程中正确防止截断，通常是通过持有 folio 锁或其他同步原语。