```c
/*
 * set_page_dirty 系列的辅助函数。
 *
 * 注意：这依赖于相对于中断的原子性。
 */
static void folio_account_dirtied(struct folio *folio,
		struct address_space *mapping)
{
	struct inode *inode = mapping->host;

	trace_writeback_dirty_folio(folio, mapping);

	if (mapping_can_writeback(mapping)) {
		struct bdi_writeback *wb;
		long nr = folio_nr_pages(folio);

		inode_attach_wb(inode, folio);
		wb = inode_to_wb(inode);

		__lruvec_stat_mod_folio(folio, NR_FILE_DIRTY, nr);
		__zone_stat_mod_folio(folio, NR_ZONE_WRITE_PENDING, nr);
		__node_stat_mod_folio(folio, NR_DIRTIED, nr);
		wb_stat_mod(wb, WB_RECLAIMABLE, nr);
		wb_stat_mod(wb, WB_DIRTIED, nr);
		task_io_account_write(nr * PAGE_SIZE);
		current->nr_dirtied += nr;
		__this_cpu_add(bdp_ratelimits, nr);

		mem_cgroup_track_foreign_dirty(folio, wb);
	}
}
```

* **目的:** 当 folio 被标记为脏时，执行记账和统计信息更新。此函数由 `__folio_mark_dirty` 调用。
* **参数:**
    * `folio`: `folio`。
    * `mapping`: `address_space`。
* **功能:**
    1. **获取 Inode:** 检索 `inode`。
    2. **追踪:** `trace_writeback_dirty_folio(folio, mapping)` 是另一个追踪点。
    3. **检查写回能力:** `if (mapping_can_writeback(mapping)) { ... }` 它检查 `address_space` 是否具有写回能力（大多数文件支持的映射都是如此）。
    4. **获取写回控制结构:**
        * `inode_attach_wb(inode, folio);` 确保 inode 与 `bdi_writeback` 结构（每个 BDI 的写回控制）关联。
        * `wb = inode_to_wb(inode);` 获取 inode 的 `bdi_writeback` 结构。
    5. **更新统计信息:** 一系列 `__lruvec_stat_mod_folio`、`__zone_stat_mod_folio`、`__node_stat_mod_folio` 和 `wb_stat_mod` 调用更新与脏页、待写入页面以及不同级别（LRU、zone、node、写回控制器）的写回活动相关的各种内核统计信息。这些统计信息用于监视系统内存压力、写回节流和性能分析。
    6. **任务 I/O 记账:** `task_io_account_write(nr * PAGE_SIZE);` 将写入 I/O 计入当前任务/进程。
    7. **当前进程脏页计数:** `current->nr_dirtied += nr;` 更新当前进程的脏页计数。
    8. **带宽延迟积 (BDP) 速率限制:** `__this_cpu_add(bdp_ratelimits, nr);` 更新每个 CPU 的 BDP 速率限制计数器，用于写回中的拥塞控制。
    9. **内存 Cgroup 跟踪:** `mem_cgroup_track_foreign_dirty(folio, wb);` 如果启用了内存 cgroup，则将脏 folio 跟踪到适当的内存 cgroup。

* **并发与截断安全:**
    * **假设持有 XArray 锁:** `folio_account_dirtied` 是从 `__folio_mark_dirty` 中调用的，后者已经持有 `xa_lock_irqsave(&mapping->i_pages)` 锁。因此，它在 XArray 锁的保护下运行，并且不需要为其页缓存数据结构添加自己的显式锁定。
    * **原子操作:** 统计信息更新（`__lruvec_stat_mod_folio` 等）可能使用原子操作或每个 CPU 的计数器来实现，以最大限度地减少争用并确保线程安全。