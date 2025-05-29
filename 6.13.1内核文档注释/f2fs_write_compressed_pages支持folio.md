 你说得非常对！这几个关键点澄清之后，事情清晰了很多，也确实可能让改动范围比最初设想的要小一些。

**核心理解和策略调整：**

1.  **`set_cluster_writeback` 的主导地位**：对于压缩文件，`folio_start_writeback` 的调用时机由 `set_cluster_writeback` 控制，而不是在 `f2fs_write_cache_folios` 的主循环中对每个从 `writeback_iter` 得到的 folio 进行。这意味着 `f2fs_write_cache_folios` 在处理压缩文件时，获取到 folio 后，不会立即调用 `folio_start_writeback`，而是将 folio 加入 `compress_ctx` (cc)，等待 `f2fs_write_multi_pages` -> `f2fs_write_compressed_pages` -> `set_cluster_writeback` 来统一处理。
2.  **`write_bytes_pending` 和 `iomap_folio_state` 的范围**：
    *   `iomap_folio_state` 和 `write_bytes_pending` **仅用于追踪 inode->i_mapping 中的常规 page cache folio（即 `cc->rpages` 指向的那些 folio）**。
    *   压缩后的 `cpages`（它们有自己的生命周期，从 `f2fs_compress_alloc_page` 分配，最终由 `f2fs_compress_free_page` 释放，并且通过 `compress_io_ctx` (cic) 的 `pending_pages` 追踪其 I/O）**不使用 `iomap_folio_state` 或我们讨论的 `write_bytes_pending` 机制**。
    *   这意味着，当 `rpages` 中的数据被成功压缩并准备通过 `cpages` 写出时，`rpages` 对应的 folio 的“回写”使命（从 `write_bytes_pending` 的角度看）可以认为在那时就接近完成了（或者说，其状态的清理逻辑会比较特殊）。

3.  **`f2fs_write_single_data_page` 的调用场景**：在 `f2fs_write_raw_pages` 中，它依然会被调用。如果 `f2fs_write_raw_pages` 被触发，意味着压缩失败或不适用，此时 `cc->rpages` 中的 folio 需要像普通文件一样被独立写回。这时，`f2fs_write_single_folio` (我们之前讨论的，内部调用 `f2fs_do_write_data_page` 或其变体) 就会派上用场，并且 `iomap_folio_state` 和 `write_bytes_pending` 的逻辑也需要被正确处理。

**基于以上理解，我们来逐个分析需要改动的函数和地方：**

---

**A. `f2fs_write_cache_folios` (处理压缩文件的分支)**

*   **`folio_start_writeback` 调用**:
    *   当 `is_compressed_file` 为 `true` 时，**不**在 `writeback_iter` 循环体开始处为从 `writeback_iter` 获取的 `folio_iterator_current` 调用 `folio_start_writeback`。
    *   也不需要立即管理 `ifs->write_bytes_pending` 的增加（bias）。这个 folio 的回写状态将由压缩流程（`set_cluster_writeback`）和其完成回调（`f2fs_compress_write_end_io` -> `end_page_writeback`）管理。
*   **`iomap_folio_state` (ifs) 的使用**:
    *   `ifs` 仍然需要被获取或分配，因为 `iomap_find_dirty_range` 依赖它来查找 folio 内的脏区段。
    *   但是，`atomic_inc(&ifs->write_bytes_pending)` (bias) 和 `atomic_dec_and_test` (在 folio 处理完毕后) 的逻辑，对于压缩路径下的 folio，需要重新考虑。如果 `set_cluster_writeback` 会为 `rpages` 中的每个 page（或其 folio）调用 `set_page_writeback`，那么 `PG_writeback` 标志会被设置。其清理由 `f2fs_compress_write_end_io` 中的 `end_page_writeback` 完成。
    *   **关键问题**：如果一个 large folio 的一部分被加入 `cc->rpages`，然后 `set_cluster_writeback` 标记了这些 page。当 `f2fs_compress_write_end_io` 完成时，它会为这些 page 调用 `end_page_writeback`。这是否意味着整个 large folio 的 `PG_writeback` 就清除了？`end_page_writeback` 是 page 粒度的。`folio_end_writeback` 是 folio 粒度的。
    *   **可能方案**：对于压缩路径，我们可能不需要 `ifs->write_bytes_pending` 来追踪整个 folio 的完成。`set_cluster_writeback` 和 `end_page_writeback` (via `f2fs_compress_write_end_io`) 已经构成了它们自己的回写状态管理。`f2fs_write_cache_folios` 在 `folio_unlock` 之后，对于压缩路径的 folio，可能不需要检查 `ifs->write_bytes_pending` 来调用 `folio_end_writeback`。

*   **将 folio 加入 `cc`**:
    *   当 `folio_iterator_current` 被加入 `cc` (通过 `f2fs_compress_ctx_add_folio`) 时，需要 `folio_get(folio_iterator_current)`。这个引用由 `f2fs_put_rpages_wbc` 或 `f2fs_put_rpages` 释放。
*   **`f2fs_write_multi_pages` 调用**: 保持不变。
*   **`wbc->nr_to_write` 更新**: 仍然依赖 `f2fs_write_multi_pages` 返回的 `*submitted` (原始数据页数量) 来更新。

---

**B. `f2fs_write_multi_pages` (入口函数)**

*   此函数本身逻辑不需要大改，它主要调度 `f2fs_compress_pages`、`f2fs_write_compressed_pages` 或 `f2fs_write_raw_pages`。
*   `f2fs_put_rpages_wbc(cc, wbc, bool clear_dirty, int put_page_type)`: 这个函数会遍历 `cc->rpages`。
    *   如果 `clear_dirty` 为 true (通常在压缩失败后)，它会 `redirty_page_for_writepage` 并 `unlock_page`。
    *   然后它会 `f2fs_folio_put(page_folio(cc->rpages[i]), true)` (unlock and put)。
    *   **Large Folio 影响**：如果 `cc->rpages` 中的多个 `page*` 来自同一个 large folio，`f2fs_folio_put` 会被多次调用在同一个 folio 上。`folio_put` 是安全的，但 `folio_unlock` 多次调用在同一个已解锁的 folio 上会 `WARN_ON`。
    *   **修改建议 `f2fs_put_rpages_wbc`**: 需要确保对每个 *unique folio* 只调用一次 `folio_unlock`。可以维护一个已解锁 folio 的集合，或者在遍历 `cc->rpages` 时，仅当 `page_folio(cc->rpages[i])` 与前一个不同或者首次遇到时才解锁。`folio_put` 可以正常多次调用。

---

**C. `f2fs_compress_pages` (执行压缩)**

*   `cc->rpages`: 包含指向原始 page cache folio 内的 `struct page*`。
*   `f2fs_vmap(cc->rpages, cc->cluster_size)`: 将这些 `page*` 映射到一个连续的虚拟地址空间 (`cc->rbuf`)。这对 large folio 应该是透明的，`vmap` 按 `page*` 数组工作。
*   `cc->cpages`: 分配用于存放压缩数据的临时页面。这些与 large folio 无直接关系。
*   **主要影响**：无直接的 large folio API 调用，逻辑上应该兼容。

---

**D. `f2fs_write_compressed_pages` (写出压缩数据)**

*   `set_cluster_writeback(cc)`:
    ```c
    static void set_cluster_writeback(struct compress_ctx *cc)
    {
        int i;
        for (i = 0; i < cc->cluster_size; i++) {
            if (cc->rpages[i])
                set_page_writeback(cc->rpages[i]); // Sets PG_writeback on the page
        }
    }
    ```
    *   **Large Folio 影响**：`set_page_writeback` 会设置 `PG_writeback`。如果多个 `cc->rpages[i]` 来自同一个 large folio，这个标志会在该 folio 上被（有效地）设置一次。这是期望的行为。

*   `fio.page = cc->rpages[i + 1]` 或 `fio.page = cc->rpages[i]`: `fio` 结构用于准备写 `cpages` (通过 `f2fs_outplace_write_data` 间接调用 `do_write_page` -> `f2fs_allocate_data_block` -> `f2fs_submit_page_write` 来写 `fio.compressed_page` 或 `fio.encrypted_page`)。
    *   `fio.page` 在这里有时被用来传递加密上下文的源数据页。
    *   `f2fs_outplace_write_data(&dn, &fio)`: 这里的 `fio` 的 `page` 成员是 `fio.compressed_page` 或 `fio.encrypted_page` (即一个 `cpage`)。`f2fs_allocate_data_block` 内部的 `folio_end_writeback` 如果被触发，是针对这个 `cpage` 的 folio，与 `rpages` 的 large folio 无关。这是好的。

*   `unlock_page(fio.page)`: 在循环中，`fio.page` 指向 `cc->rpages[i]`。
    *   **Large Folio 影响**：如果多个 `cc->rpages[i]` 来自同一个 large folio，`unlock_page` 会被多次调用在同一个 folio 的不同 page 上。`unlock_page` 最终会调用 `folio_unlock`。多次对已解锁的 folio 调用 `folio_unlock` 会 `WARN_ON`。
    *   **修改建议**: 与 `f2fs_put_rpages_wbc` 类似，需要确保对每个 *unique folio* (来自 `cc->rpages`) 只调用一次 `folio_unlock`。可以在循环结束后，遍历 `cc->rpages` 中的 unique folios 进行解锁。

*   `f2fs_put_rpages(cc)`: 在函数成功路径的末尾调用。
    ```c
    void f2fs_put_rpages(struct compress_ctx *cc)
    {
        int i;
        for (i = 0; i < cc->cluster_size; i++) {
            if (cc->rpages[i]) {
                // Assuming f2fs_folio_put handles unlock if page was locked by this context
                // and cc is now done with it.
                // If pages were from locked folios (e.g. Pass 2 of prepare_compress_overwrite)
                // and not unlocked yet, this is where they should be unlocked.
                // However, the loop above in f2fs_write_compressed_pages *already did unlock_page*.
                f2fs_folio_put(page_folio(cc->rpages[i]), false /* unlock=false, already unlocked */);
                cc->rpages[i] = NULL;
            }
        }
        cc->nr_rpages = 0;
    }
    ```
    *   **修改建议**: `f2fs_put_rpages` 应该只负责 `folio_put`。解锁操作应该在 `f2fs_write_compressed_pages` 的循环中或者循环后统一处理 unique folios。如果 `f2fs_write_compressed_pages` 的循环已经解锁了，那么这里的 `f2fs_folio_put` 的 `unlock` 参数应为 `false`。

*   `cancel_cluster_writeback(cc, cic, submitted)`:
    *   `end_page_writeback(cc->rpages[i])`: 清理 `PG_writeback`。对于 large folio，如果多个 `rpages` 来自同一个 folio，这会多次作用于同一个 folio 的 `PG_writeback`，但 `end_page_writeback` 最终会调用 `folio_end_writeback`，它有自己的保护。
    *   `lock_page(cc->rpages[i])`: 如果之前被解锁了，这里重新锁定。同样需要注意 unique folio。

---

**E. `f2fs_compress_write_end_io` (压缩IO完成回调)**

*   `struct page *page`: 这是传入的 `cpage`（压缩数据页）。
*   `cic->rpages`: 指向原始数据页（可能在 large folio 中）。
*   `end_page_writeback(cic->rpages[i])`:
    *   对每个原始数据页（`rpage`）调用 `end_page_writeback`。
    *   `end_page_writeback` 会调用 `folio_end_writeback(page_folio(cic->rpages[i]))`。
    *   **Large Folio 影响**：如果 `cic->rpages` 中的多个 `page*` 来自同一个 large folio，`folio_end_writeback` 会被多次调用在该 folio 上。`folio_end_writeback` 内部有 `if (!folio_trylock(folio)) return;` 和 `folio_unlock(folio)`，并且通过 `test_and_clear_bit(PG_writeback, &folio->flags)` 来确保 `PG_writeback` 只被有效清除一次，并唤醒等待者。所以多次调用 `folio_end_writeback` 应该是安全的，尽管可能不是最高效的。理想情况下，对一个 large folio，`folio_end_writeback` 只被调用一次。
    *   **思考**：由于 `cic->pending_pages` 追踪的是 `cpages` 的完成，当所有 `cpages` 完成时，这个回调被触发。此时，所有相关的 `rpages` 的回写也应视为完成。我们是否可以在这里，当 `atomic_dec_return(&cic->pending_pages)` 为0时，遍历 `cic->rpages`，收集所有 *unique folios*，然后对每个 unique folio 调用一次 `folio_end_writeback`？这会更精确。

---

**F. `f2fs_write_raw_pages` (压缩失败或不适用时的回退路径)**

*   这个函数遍历 `cc->rpages`，对每个 page 调用 `redirty_page_for_writepage`，`unlock_page`。
*   然后再次循环，`folio_lock`，并调用 `f2fs_write_single_data_page`。
*   **Large Folio 影响**:
    *   **解锁/加锁**: 多次在同一个 large folio 的不同 page 上调用 `unlock_page` 后再 `folio_lock`。需要确保 `folio_lock` 是针对 unique folio 操作的。
    *   **`f2fs_write_single_data_page`**: 我们已经计划将其替换为 `f2fs_write_single_folio` (或其内部逻辑)。
        *   `ret = f2fs_write_single_folio(page_folio(cc->rpages[i]), &submitted_one_folio, ..., wbc, ..., dirty_start_offset, dirty_len);`
        *   这里需要确定 `dirty_start_offset` 和 `dirty_len`。由于 `f2fs_write_raw_pages` 是按 cluster 处理的，并且 `cc->rpages[i]` 是其中的一个 page，那么对于这个 `f2fs_write_single_folio` 调用，它实际上是针对 `cc->rpages[i]` 这个单页的范围。
        *   **因此，这里调用 `f2fs_write_single_folio` 传递的范围应该是单个 page 的范围。**
        *   `f2fs_write_single_folio` 内部会处理 `ifs->write_bytes_pending` 的增加和减少（在错误时）。
        *   `folio_start_writeback` 应该在 `f2fs_write_raw_pages` 的第二次循环中，在 `folio_lock` 之后，为每个 *unique folio* 调用一次。
        *   `folio_end_writeback` 将由 `f2fs_write_single_folio` 内部的 `f2fs_write_end_io` (成功路径) 或 `f2fs_write_single_folio` 的错误处理路径 (结合 `ifs` 和 bias) 来处理。

**修改建议 `f2fs_write_raw_pages`**:

```c
static int f2fs_write_raw_pages(struct compress_ctx *cc,
					int *submitted_p,
					struct writeback_control *wbc,
					enum iostat_type io_type)
{
	struct address_space *mapping = cc->inode->i_mapping;
	struct f2fs_sb_info *sbi = F2FS_M_SB(mapping);
	int compr_blocks;
	int ret = 0;
    struct folio *current_processing_folio = NULL;
    pgoff_t last_processed_folio_idx = NULL_पृष्ठ; // Sentinel for page index

	compr_blocks = f2fs_compressed_blocks(cc); // This seems to be about previous state

    // First loop: redirty and unlock pages from cc->rpages.
    // This prepares them for individual writeback if they were part of a compression attempt.
	for (int i = 0; i < cc->cluster_size; i++) {
		struct page *page = cc->rpages[i];
		struct folio *folio;
		if (!page)
			continue;

		folio = page_folio(page);
		// Ensure we only unlock each unique folio once if it was locked.
		// However, pages in cc->rpages are expected to be from locked folios
		// (from prepare_compress_overwrite Pass 2).
		// The original code unlocks each page. This implies each page was individually locked,
		// or it's safe to call unlock_page on pages of an already unlocked folio (not true).
		// This loop needs to be folio-aware for unlocking.
        // For now, let's assume a conceptual unique folio unlock or that
        // these pages are targets and their folios are managed.
        // The original code did unlock_page(cc->rpages[i]).
        // If multiple rpages are from same folio, this is problematic.
        // Let's assume for now this part is made folio-safe externally or by caller.
        // A simple fix: only unlock if it's a new folio.
        if (page_folio(cc->rpages[i]) != current_processing_folio) {
            if (current_processing_folio && folio_test_locked(current_processing_folio)) {
                folio_unlock(current_processing_folio);
            }
            current_processing_folio = page_folio(cc->rpages[i]);
        }
		redirty_page_for_writepage(wbc, cc->rpages[i]);
	}
    if (current_processing_folio && folio_test_locked(current_processing_folio)) { // Unlock the last one
        folio_unlock(current_processing_folio);
    }
    current_processing_folio = NULL; // Reset for next loop


	if (compr_blocks < 0) // Error from f2fs_compressed_blocks
		return compr_blocks;

	/* overwrite compressed cluster w/ normal cluster */
	if (compr_blocks > 0)
		f2fs_lock_op(sbi);

    // Second loop: lock and write each page individually using f2fs_write_single_folio
	for (int i = 0; i < cc->cluster_size; i++) {
		struct page *page = cc->rpages[i];
		struct folio *folio;
        loff_t page_pos_in_file;
        int submitted_this_page = 0; // For f2fs_write_single_folio

		if (!page)
			continue;

		folio = page_folio(page);
        page_pos_in_file = folio_pos(folio) + (folio_page_idx(folio, page) << PAGE_SHIFT);

		// Lock the folio for this page.
		// If current_processing_folio is already this folio and locked, skip locking.
		if (folio != current_processing_folio) {
            if (current_processing_folio && folio_test_locked(current_processing_folio)) {
                // This should not happen if logic is correct, implies previous folio not handled.
                // Or, this is where we'd handle the end of processing for the previous folio.
                // For simplicity, assume we lock each folio as we encounter its first page from rpages.
            }
            folio_lock(folio);
            current_processing_folio = folio;
            // Set PG_writeback for this folio if not already set by a broader context.
            // This is tricky because f2fs_write_cache_folios might have done it.
            // If f2fs_write_raw_pages is called, the compression path failed.
            // The folio from writeback_iter might have had folio_start_writeback called.
            // Let's assume it was. If not, it needs to be called here for current_processing_folio.
            if (!folio_test_writeback(current_processing_folio)) {
                 folio_start_writeback(current_processing_folio);
            }
        }


		if (folio->mapping != mapping) {
			// Folio moved or truncated.
            if (folio_test_locked(folio)) folio_unlock(folio); // Unlock if we locked it
            current_processing_folio = NULL;
			continue;
		}

		if (!folio_test_dirty(folio)) { // Check dirty for the whole folio
			if (folio_test_locked(folio)) folio_unlock(folio);
            current_processing_folio = NULL;
			continue;
		}

        // PG_writeback should be set now (either by f2fs_write_cache_folios or just above)
		if (folio_test_writeback(folio) && folio != current_processing_folio /* or some other condition to re-check */) {
            // This means another thread is writing this folio.
            // If wbc->sync_mode == WB_SYNC_NONE, we skip. Otherwise, wait.
			if (wbc->sync_mode == WB_SYNC_NONE) {
				if (folio_test_locked(folio)) folio_unlock(folio);
                current_processing_folio = NULL;
				continue;
			}
			f2fs_folio_wait_writeback(folio, DATA, true, true); // Waits on PG_writeback
		}

        // folio_clear_dirty_for_io operates on the whole folio.
		if (!folio_clear_dirty_for_io(folio)) {
			if (folio_test_locked(folio)) folio_unlock(folio);
            current_processing_folio = NULL;
			continue;
		}

        // Call f2fs_write_single_folio for the range of this single page
        // f2fs_write_single_folio will handle ifs->write_bytes_pending internally.
		ret = f2fs_write_single_folio(folio, &submitted_this_page,
						NULL, NULL, /* bio, last_block */
                        wbc, io_type,
						0, /* compr_blocks for raw write */
                        false /* is_reclaim */,
                        page_pos_in_file, /* start of this page */
                        page_pos_in_file + PAGE_SIZE /* end of this page */);

		*submitted_p += submitted_this_page; // Accumulate submitted pages

		if (ret) {
			// f2fs_write_single_folio returns 0 on success, -EAGAIN, or other errors.
			// It handles mapping_set_error.
			// If AOP_WRITEPAGE_ACTIVATE was the old model, f2fs_write_single_folio
			// doesn't return that; lock management is external.
			// If -EAGAIN, it means wbc->nr_to_write hit 0 or internal retry needed.
			if (ret == -EAGAIN) { // wbc budget met or retry suggested by f2fs_do_write_data_page
				// The original code had a retry_write for -EAGAIN from f2fs_write_single_data_page
				// and specific handling for quota.
				// For now, if -EAGAIN, we assume f2fs_write_single_folio handled redirty if needed.
				// We should break from this loop and let writeback_iter decide.
                // No, original code retried for non-quota.
                // This retry logic needs to be carefully preserved if f2fs_write_single_folio
                // can return -EAGAIN for reasons other than wbc->nr_to_write == 0.
                // For now, assume -EAGAIN means stop this folio or cluster.
				ret = 0; // Clear -EAGAIN for outer loop if it was just budget.
			}
            // On other errors, f2fs_write_single_folio's error handling for ifs->write_bytes_pending
            // and mapping_set_error should have occurred.
            // We need to unlock the current_processing_folio here before breaking/continuing.
            if (folio_test_locked(current_processing_folio)) {
                folio_unlock(current_processing_folio);
            }
            current_processing_folio = NULL;
			if (ret != 0) goto out; // Break on real error
		}
        // If this was the last page of current_processing_folio being processed from rpages,
        // or if the next rpages[i+1] is from a different folio, unlock current_processing_folio.
        if (i + 1 == cc->cluster_size || !cc->rpages[i+1] || page_folio(cc->rpages[i+1]) != current_processing_folio) {
            if (current_processing_folio && folio_test_locked(current_processing_folio)) {
                folio_unlock(current_processing_folio);
            }
            current_processing_folio = NULL;
        }
	} // End of for loop iterating cc->rpages

out:
    if (current_processing_folio && folio_test_locked(current_processing_folio)) { // Ensure last folio is unlocked
        folio_unlock(current_processing_folio);
    }
	if (compr_blocks > 0)
		f2fs_unlock_op(sbi);

	f2fs_balance_fs(sbi, true);
	return ret;
}
```

**总结改动方向：**

1.  **`f2fs_write_cache_folios`**:
    *   压缩路径：不主动调 `folio_start_writeback` 或管理 `ifs->write_bytes_pending` (bias part)。依赖 `set_cluster_writeback` 和 `f2fs_compress_write_end_io`。
    *   非压缩路径：使用 `f2fs_write_single_folio`，它内部管理 `ifs->write_bytes_pending`。`folio_start_writeback` 在 `writeback_iter` 循环开始时（或 `f2fs_write_raw_pages` 内部）调用。
2.  **`f2fs_put_rpages_wbc` / `f2fs_write_compressed_pages` (unlocking `rpages`)**: 确保对 unique folios 只 unlock 一次。
3.  **`f2fs_compress_write_end_io`**: 考虑对 unique folios 调用 `folio_end_writeback` 一次，而不是通过 `end_page_writeback` 多次。
4.  **`f2fs_write_raw_pages`**:
    *   改为调用 `f2fs_write_single_folio` 处理每个 page 的范围。
    *   正确管理 `folio_lock` 和 `folio_start_writeback` 对于从 `cc->rpages` 中提取的 unique folios。
    *   `ifs->write_bytes_pending` 由 `f2fs_write_single_folio` 处理。

这个分析应该覆盖了主要的 large folio 交互点。下一步是细化这些函数中的 folio 迭代和状态管理逻辑。



