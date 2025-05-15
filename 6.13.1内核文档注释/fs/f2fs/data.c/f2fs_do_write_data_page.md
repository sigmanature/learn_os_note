 [f2fs_write_single_data_page](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/data.c/f2fs_write_single_data_page.md)

**1. `f2fs_do_write_data_page(struct f2fs_io_info *fio)` 函数详解**

我们先来详细解析 `f2fs_do_write_data_page` 函数，理解其在数据块写回流程中的作用和逻辑：

```c
int f2fs_do_write_data_page(struct f2fs_io_info *fio)
{
	struct folio *folio = page_folio(fio->page);
	struct inode *inode = folio->mapping->host;
	struct dnode_of_data dn;
	struct node_info ni;
	bool ipu_force = false;
	bool atomic_commit;
	int err = 0;

	/* Use COW inode to make dnode_of_data for atomic write */
	atomic_commit = f2fs_is_atomic_file(inode) &&
				page_private_atomic(folio_page(folio, 0));
	if (atomic_commit)
		set_new_dnode(&dn, F2FS_I(inode)->cow_inode, NULL, NULL, 0);
	else
		set_new_dnode(&dn, inode, NULL, NULL, 0);

	if (need_inplace_update(fio) &&
	    f2fs_lookup_read_extent_cache_block(inode, folio->index,
						&fio->old_blkaddr)) {
		if (!f2fs_is_valid_blkaddr(fio->sbi, fio->old_blkaddr,
						DATA_GENERIC_ENHANCE))
			return -EFSCORRUPTED;

		ipu_force = true;
		fio->need_lock = LOCK_DONE;
		goto got_it;
	}

	/* Deadlock due to between page->lock and f2fs_lock_op */
	if (fio->need_lock == LOCK_REQ && !f2fs_trylock_op(fio->sbi))
		return -EAGAIN;

	err = f2fs_get_dnode_of_data(&dn, folio->index, LOOKUP_NODE);
	if (err)
		goto out;

	fio->old_blkaddr = dn.data_blkaddr;

	/* This page is already truncated */
	if (fio->old_blkaddr == NULL_ADDR) {
		folio_clear_uptodate(folio);
		clear_page_private_gcing(folio_page(folio, 0));
		goto out_writepage;
	}
  got_it:
	if (__is_valid_data_blkaddr(fio->old_blkaddr) &&
		!f2fs_is_valid_blkaddr(fio->sbi, fio->old_blkaddr,
						DATA_GENERIC_ENHANCE)) {
		err = -EFSCORRUPTED;
		goto out_writepage;
	}

	/* wait for GCed page writeback via META_MAPPING */
	if (fio->meta_gc)
		f2fs_wait_on_block_writeback(inode, fio->old_blkaddr);

	/*
	 * If current allocation needs SSR,
	 * it had better in-place writes for updated data.
	 */
	if (ipu_force ||
		(__is_valid_data_blkaddr(fio->old_blkaddr) &&
					need_inplace_update(fio))) {
		err = f2fs_encrypt_one_page(fio);
		if (err)
			goto out_writepage;

		folio_start_writeback(folio);
		f2fs_put_dnode(&dn);
		if (fio->need_lock == LOCK_REQ)
			f2fs_unlock_op(fio->sbi);
		err = f2fs_inplace_write_data(fio);
		if (err) {
			if (fscrypt_inode_uses_fs_layer_crypto(inode))
				fscrypt_finalize_bounce_page(&fio->encrypted_page);
			folio_end_writeback(folio);
		} else {
			set_inode_flag(inode, FI_UPDATE_WRITE);
		}
		trace_f2fs_do_write_data_page(folio, IPU);
		return err;
	}

	if (fio->need_lock == LOCK_RETRY) {
		if (!f2fs_trylock_op(fio->sbi)) {
			err = -EAGAIN;
			goto out_writepage;
		}
		fio->need_lock = LOCK_REQ;
	}

	err = f2fs_get_node_info(fio->sbi, dn.nid, &ni, false);
	if (err)
		goto out_writepage;

	fio->version = ni.version;

	err = f2fs_encrypt_one_page(fio);
	if (err)
		goto out_writepage;

	folio_start_writeback(folio);

	if (fio->compr_blocks && fio->old_blkaddr == COMPRESS_ADDR)
		f2fs_i_compr_blocks_update(inode, fio->compr_blocks - 1, false);

	/* LFS mode write path */
	f2fs_outplace_write_data(&dn, fio);
	trace_f2fs_do_write_data_page(folio, OPU);
	set_inode_flag(inode, FI_APPEND_WRITE);
	if (atomic_commit)
		clear_page_private_atomic(folio_page(folio, 0));
out_writepage:
	f2fs_put_dnode(&dn);
out:
	if (fio->need_lock == LOCK_REQ)
		f2fs_unlock_op(fio->sbi);
	return err;
}
```

*   **功能:** `f2fs_do_write_data_page` 函数是 F2FS 中 **数据页面写回的核心函数**。  **无论是 Page Cache 的脏页回写，还是 GC 数据迁移，最终都会调用 `f2fs_do_write_data_page` 函数将数据页面写入存储设备。**  `f2fs_do_write_data_page` 函数负责处理 Inplace Update (IPU) 和 Outplace Update (OPU) 两种写回路径，并进行加密、压缩、IO 提交等操作。

*   **参数:**  `struct f2fs_io_info *fio`:  `f2fs_io_info` 结构体指针，描述了写 IO 操作的信息，包括页面、Inode、IO 类型、操作标志等。

*   **变量初始化:**  初始化局部变量 `folio`, `inode`, `dn`, `ni`, `ipu_force`, `atomic_commit`, `err`。

*   **Atomic Commit 和 COW Inode 处理:**
    ```c
    /* Use COW inode to make dnode_of_data for atomic write */
    atomic_commit = f2fs_is_atomic_file(inode) &&
                page_private_atomic(folio_page(folio, 0));
    if (atomic_commit)
        set_new_dnode(&dn, F2FS_I(inode)->cow_inode, NULL, NULL, 0);
    else
        set_new_dnode(&dn, inode, NULL, NULL, 0);
    ```
    *   **检查是否需要 Atomic Commit:**  `atomic_commit = f2fs_is_atomic_file(inode) && page_private_atomic(folio_page(folio, 0));`:  判断是否需要进行原子提交 (Atomic Commit)。  条件包括：文件是否是原子文件 (`f2fs_is_atomic_file(inode)`)，以及页面是否设置了原子私有标志 (`page_private_atomic(folio_page(folio, 0))`)。  **Atomic Commit 用于保证写操作的原子性，即使在系统崩溃的情况下也能保证数据一致性。**
    *   **选择 Dnode 的 Inode:**  `if (atomic_commit) set_new_dnode(&dn, F2FS_I(inode)->cow_inode, NULL, NULL, 0); else set_new_dnode(&dn, inode, NULL, NULL, 0);`:  **如果需要 Atomic Commit，则使用 COW Inode (`F2FS_I(inode)->cow_inode`) 创建 `dnode_of_data` 结构体 `dn`，否则使用原始 Inode (`inode`)**。  **COW Inode 用于存储原子写的新版本数据。**

*   **Inplace Update (IPU) 快速路径检查:**
    ```c
    if (need_inplace_update(fio) &&
        f2fs_lookup_read_extent_cache_block(inode, folio->index,
                        &fio->old_blkaddr)) {
        if (!f2fs_is_valid_blkaddr(fio->sbi, fio->old_blkaddr,
                        DATA_GENERIC_ENHANCE))
            return -EFSCORRUPTED;

        ipu_force = true;
        fio->need_lock = LOCK_DONE;
        goto got_it;
    }
    ```
    *   **检查是否需要 Inplace Update (`need_inplace_update(fio)`)**。  Inplace Update 指的是 **在 *原始位置* 直接更新数据块**，而不是分配新的块地址进行 Outplace Update。  Inplace Update 通常用于 **小范围的数据更新，可以减少写放大和元数据更新开销**。
    *   **Extent Cache 查找旧块地址:**  `f2fs_lookup_read_extent_cache_block(inode, folio->index, &fio->old_blkaddr)`:  **尝试从 extent cache 中查找数据块的 *旧块地址* (`fio->old_blkaddr`)**。  Inplace Update 需要知道旧块地址，才能在原位置更新数据。
    *   **旧块地址有效性检查:**  `if (!f2fs_is_valid_blkaddr(fio->sbi, fio->old_blkaddr, DATA_GENERIC_ENHANCE)) return -EFSCORRUPTED;`:  **检查从 extent cache 获取的旧块地址是否有效**。  如果无效，则返回 `-EFSCORRUPTED` 错误。
    *   **设置 `ipu_force` 和 `fio->need_lock`:**  `ipu_force = true; fio->need_lock = LOCK_DONE;`:  **设置 `ipu_force` 标志为真，表示强制使用 Inplace Update 路径**。  设置 `fio->need_lock = LOCK_DONE;`，表示不需要获取操作锁 (因为已经通过 extent cache 找到了旧块地址，可以安全地进行 Inplace Update)。
    *   **`goto got_it;`:**  **跳转到 `got_it` 标签，跳过 dnode 查找，直接进入 Inplace Update 路径。**

*   **操作锁尝试获取 (避免死锁):**
    ```c
    /* Deadlock due to between page->lock and f2fs_lock_op */
    if (fio->need_lock == LOCK_REQ && !f2fs_trylock_op(fio->sbi))
        return -EAGAIN;
    ```
    *   **检查是否需要获取操作锁 (`fio->need_lock == LOCK_REQ`)**。  操作锁 (`f2fs_lock_op`) 用于 **保护文件系统的某些全局操作的并发安全**。
    *   **尝试非阻塞获取操作锁:**  `if (fio->need_lock == LOCK_REQ && !f2fs_trylock_op(fio->sbi)) return -EAGAIN;`:  **尝试非阻塞地获取操作锁 (`f2fs_trylock_op`)**。  如果获取失败，则返回 `-EAGAIN` 错误，表示稍后重试。  **避免在页面锁和操作锁之间发生死锁。**

*   **Dnode 查找 (获取旧块地址):**
    ```c
    err = f2fs_get_dnode_of_data(&dn, folio->index, LOOKUP_NODE);
    if (err)
        goto out;

    fio->old_blkaddr = dn.data_blkaddr;
    ```
    *   **调用 `f2fs_get_dnode_of_data` 函数，根据 `folio->index` (逻辑块索引) 在 dnode 树中查找数据块的 dnode 信息，并获取数据块的 *旧块地址* (`dn.data_blkaddr`)**。  **这是获取数据块旧块地址的主要路径 (如果 Inplace Update 快速路径未命中)。**

*   **Truncated 页面检查:**
    ```c
    /* This page is already truncated */
    if (fio->old_blkaddr == NULL_ADDR) {
        folio_clear_uptodate(folio);
        clear_page_private_gcing(folio_page(folio, 0));
        goto out_writepage;
    }
    ```
    *   **检查旧块地址是否为 `NULL_ADDR`**。  如果是，表示页面已被截断 (truncated)。
    *   **截断页面处理:**  清除页面 uptodate 标志，清除页面 GC 标记，并跳转到 `out_writepage` 标签，跳过写操作。  **对于截断页面，无需进行写回操作。**

*   **`got_it:` 标签和旧块地址有效性检查:**
    ```c
  got_it:
	if (__is_valid_data_blkaddr(fio->old_blkaddr) &&
		!f2fs_is_valid_blkaddr(fio->sbi, fio->old_blkaddr,
						DATA_GENERIC_ENHANCE)) {
		err = -EFSCORRUPTED;
		goto out_writepage;
	}
    ```
    *   **`got_it:` 标签:**  Inplace Update 快速路径和 dnode 查找路径都会到达 `got_it` 标签。
    *   **旧块地址有效性检查:**  **再次检查旧块地址 `fio->old_blkaddr` 的有效性**。  确保旧块地址在有效的数据块地址范围内。  如果无效，则返回 `-EFSCORRUPTED` 错误。

*   **等待 GCed 页面 Writeback (Meta GC):**
    ```c
    /* wait for GCed page writeback via META_MAPPING */
    if (fio->meta_gc)
        f2fs_wait_on_block_writeback(inode, fio->old_blkaddr);
    ```
    *   **检查是否为 Meta GC (`fio->meta_gc`)**。  Meta GC 指的是针对元数据页面的 GC 操作。
    *   **等待旧块地址的 Writeback:**  如果是 Meta GC，则 **等待旧块地址 `fio->old_blkaddr` 上的 Writeback 操作完成**。  **避免 Meta GC 和普通 GC 之间的竞争。**

*   **Inplace Update (IPU) 路径:**
    ```c
    /*
     * If current allocation needs SSR,
     * it had better in-place writes for updated data.
     */
    if (ipu_force ||
        (__is_valid_data_blkaddr(fio->old_blkaddr) &&
                    need_inplace_update(fio))) {
        err = f2fs_encrypt_one_page(fio);
        if (err)
            goto out_writepage;

        folio_start_writeback(folio);
        f2fs_put_dnode(&dn);
        if (fio->need_lock == LOCK_REQ)
            f2fs_unlock_op(fio->sbi);
        err = f2fs_inplace_write_data(fio);
        if (err) {
            if (fscrypt_inode_uses_fs_layer_crypto(inode))
                fscrypt_finalize_bounce_page(&fio->encrypted_page);
            folio_end_writeback(folio);
        } else {
            set_inode_flag(inode, FI_UPDATE_WRITE);
        }
        trace_f2fs_do_write_data_page(folio, IPU);
        return err;
    }
    ```
    *   **再次检查是否需要 Inplace Update (`ipu_force || (__is_valid_data_blkaddr(fio->old_blkaddr) && need_inplace_update(fio))`)**。  条件包括：`ipu_force` 标志为真 (快速路径命中)，或者旧块地址有效且需要 Inplace Update。
    *   **页面加密:**  `err = f2fs_encrypt_one_page(fio);`:  **对页面数据进行加密 (如果需要)**。
    *   **启动页面 Writeback:**  `folio_start_writeback(folio);`:  **启动页面的 Writeback 流程**。
    *   **释放 Dnode 和操作锁:**  `f2fs_put_dnode(&dn); if (fio->need_lock == LOCK_REQ) f2fs_unlock_op(fio->sbi);`:  释放 `dnode_of_data` 结构体和操作锁 (如果已获取)。
    *   **执行 Inplace Write:**  `err = f2fs_inplace_write_data(fio);`:  **调用 `f2fs_inplace_write_data` 函数执行 *Inplace Write* 操作，将页面数据 *直接写入到旧块地址*。**
    *   **错误处理和 Writeback 结束:**  如果 `f2fs_inplace_write_data` 返回错误，则进行错误处理 (例如，释放加密 bounce page, 结束 Writeback)。  否则，设置 Inode 的 `FI_UPDATE_WRITE` 标志。
    *   **Tracepoint 记录:**  `trace_f2fs_do_write_data_page(folio, IPU);`:  记录 Inplace Update 路径的 tracepoint 信息。
    *   **返回错误码:**  `return err;`:  返回错误码。

*   **操作锁重试 (Lock Retry):**
    ```c
    if (fio->need_lock == LOCK_RETRY) {
        if (!f2fs_trylock_op(fio->sbi)) {
            err = -EAGAIN;
            goto out_writepage;
        }
        fio->need_lock = LOCK_REQ;
    }
    ```
    *   **检查是否需要重试获取操作锁 (`fio->need_lock == LOCK_RETRY`)**。  `LOCK_RETRY` 可能表示之前尝试获取操作锁失败，需要重试。
    *   **再次尝试非阻塞获取操作锁:**  `if (!f2fs_trylock_op(fio->sbi)) { err = -EAGAIN; goto out_writepage; }`:  **再次尝试非阻塞地获取操作锁**。  如果仍然失败，则返回 `-EAGAIN` 错误，跳转到 `out_writepage` 标签。
    *   **设置 `fio->need_lock = LOCK_REQ;`:**  如果成功获取操作锁，则设置 `fio->need_lock = LOCK_REQ;`，表示已获取操作锁。

*   **获取 Node Info (Outplace Update):**
    ```c
    err = f2fs_get_node_info(fio->sbi, dn.nid, &ni, false);
    if (err)
        goto out_writepage;

    fio->version = ni.version;
    ```
    *   **调用 `f2fs_get_node_info` 函数，根据 `dn.nid` (dnode 的 NID) 获取 Node Info 结构体 `ni`**。  **Outplace Update 路径需要获取 Node Info，可能用于更新父节点信息或 NAT 表。**
    *   **保存 Node 版本号:**  `fio->version = ni.version;`:  **保存 Node 版本号 `ni.version` 到 `fio->version` 字段**。  版本号可能用于并发控制和数据一致性。

*   **页面加密 (Outplace Update):**  `err = f2fs_encrypt_one_page(fio);`:  **对页面数据进行加密 (如果需要)**。

*   **启动页面 Writeback (Outplace Update):**  `folio_start_writeback(folio);`:  **启动页面的 Writeback 流程**。

*   **压缩块计数更新 (如果需要):**
    ```c
    if (fio->compr_blocks && fio->old_blkaddr == COMPRESS_ADDR)
        f2fs_i_compr_blocks_update(inode, fio->compr_blocks - 1, false);
    ```
    *   **检查是否需要更新压缩块计数 (`fio->compr_blocks && fio->old_blkaddr == COMPRESS_ADDR`)**。  如果需要，则调用 `f2fs_i_compr_blocks_update` 函数 **更新 Inode 的压缩块计数器**。  **这部分代码可能与 F2FS 的压缩特性有关。**

*   **Outplace Update (OPU) 路径:**  `f2fs_outplace_write_data(&dn, fio);`:  **调用 `f2fs_outplace_write_data` 函数执行 *Outplace Write* 操作，将页面数据 *写入到新分配的块地址* (由 `f2fs_outplace_write_data` 函数内部完成块分配和 IO 提交)。**  **Outplace Update 是 F2FS 的 *默认写回路径*，也是 LFS (Log-structured File System) 的核心思想。**
*   **Tracepoint 记录:**  `trace_f2fs_do_write_data_page(folio, OPU);`:  记录 Outplace Update 路径的 tracepoint 信息.
*   **设置 Inode 标志和清除 Atomic 标志:**  `set_inode_flag(inode, FI_APPEND_WRITE); if (atomic_commit) clear_page_private_atomic(folio_page(folio, 0));`:  设置 Inode 的 `FI_APPEND_WRITE` 标志，并清除页面的原子私有标志 (如果之前设置了 Atomic Commit)。

*   **`out_writepage:` 标签和 Dnode 释放:**  `f2fs_put_dnode(&dn);`:  **释放 `dnode_of_data` 结构体 `dn`**。

*   **`out:` 标签和操作锁释放:**  `if (fio->need_lock == LOCK_REQ) f2fs_unlock_op(fio->sbi);`:  **释放操作锁 (如果之前获取了操作锁)**。

*   **返回值:**  `return err;`:  **返回错误码**。

*   **总结 `f2fs_do_write_data_page`:**  `f2fs_do_write_data_page` 函数是 F2FS **数据页面写回的核心控制中心**。  它 **根据不同的条件 (例如，Inplace Update, Atomic Commit, GC 类型等)，选择不同的写回路径 (Inplace Update 或 Outplace Update)**，并负责 **页面加密、操作锁管理、IO 提交、元数据更新和错误处理** 等一系列操作。  **`f2fs_do_write_data_page` 函数的实现非常复杂，体现了 F2FS 在性能、可靠性和数据一致性方面的综合考虑。**

**2. 数据块逻辑 ID 映射关系和 Page Cache 写回**

你提出的 "数据块的逻辑 ID 不直接存储在数据块自己中" 的观察 **非常敏锐且重要！**  **F2FS 的数据块 *本身* 确实 *不存储* 任何逻辑 ID 信息**。  **数据块的逻辑 ID (例如，在 Inode 地址空间中的块索引) 是 *隐式* 地通过 Inode 的元数据结构 (extent cache, dnode 树) 来维护和管理的。**

*   **Page Cache 写回的挑战:**  **在 Page Cache 写回场景下，我们只有 Folio (或 Page) 结构体，以及 Folio 的 *连续逻辑索引* (我们为了 Folio 优化而引入的)。  我们 *没有* 直接的数据块逻辑 ID (例如，`f2fs_start_bidx_of_node` 计算出的原始索引) 可以直接使用。**  **`f2fs_do_write_data_page` 函数仍然需要 *原始的逻辑索引* (例如，`folio->index`) 来进行 extent cache 和 dnode 查找，以获取旧块地址和进行后续的写回操作。**

*   **内存映射表的必要性 (临时性映射):**  为了在 Page Cache 写回场景下，将 Folio 的 *连续逻辑索引* 转换为 `f2fs_do_write_data_page` 函数所需的 *原始逻辑索引*，**我们需要在内存中维护一个 *临时的映射表***。  **这个映射表只需要在 Page Cache 写回过程中 *临时存在*，不需要持久化到磁盘。**

*   **映射表的设计 (初步想法):**  我们可以考虑使用一个 **简单的数组或哈希表** 来实现这个映射表。  **映射表的 Key 是 Folio 的 *连续逻辑索引*，Value 是 *原始逻辑索引* (例如，`f2fs_start_bidx_of_node` 计算出的索引)。**

    *   **映射表填充**:  在 Page Cache 写回之前，我们需要 **遍历 Folio 中的所有页面，并为每个页面 *计算* 其 *原始逻辑索引* (例如，通过 `f2fs_start_bidx_of_node` 函数)，然后将 *连续逻辑索引* 和 *原始逻辑索引* 的映射关系 *添加到映射表中***。
    *   **`f2fs_do_write_data_page` 中索引转换**:  在 `f2fs_do_write_data_page` 函数中，**不再直接使用 `folio->index`，而是 *首先* 使用 `folio->index` (连续逻辑索引) 在映射表中 *查找* 对应的 *原始逻辑索引*，然后使用 *原始逻辑索引* 进行后续的 extent cache 和 dnode 查找操作。**
    *   **映射表释放**:  在 Page Cache 写回完成后，**释放映射表，回收内存**。

*   **映射表的持久化 (无需持久化):**  你说的 "这个映射表能不存到磁盘的话就别存到磁盘上" **完全正确！**  **这个映射表 *没有持久化的必要*，只需要在内存中临时存在即可。**  **持久化映射表会大大增加复杂性，并可能引入新的性能开销，而且没有实际意义。**  **Page Cache 的本质就是内存缓存，其索引和映射关系都是 *临时的*，不需要持久化。**

**3. 改动影响范围和回写操作研读**

你提出的 "必须同时考虑我们的改动对所有其他回写操作的影响，意味着我们也同时需要仔细研读其他的回写操作" **非常重要！**  **`f2fs_do_write_data_page` 函数是 F2FS 数据写回的 *通用入口点*，任何修改都可能影响到 F2FS 的 *所有写回路径***，包括：

*   **Page Cache 脏页回写 (background writeback)**
*   **`fsync` 系统调用触发的同步写回**
*   **内存压力导致的页面回收 (memory reclaim)**
*   **GC 数据迁移 (garbage collection)**
*   **Direct IO 写操作 (bypass Page Cache)** (虽然 Direct IO 通常 bypass Page Cache，但某些 Direct IO 实现可能仍然会涉及到 Page Cache 的部分机制)

**我们需要仔细研读 F2FS 的 *所有写回路径*，分析 `f2fs_do_write_data_page` 函数在不同写回路径下的调用方式和参数，评估我们的 Folio 优化方案对 *所有写回路径* 的影响，确保我们的改动 *不会破坏 F2FS 的数据一致性和可靠性*，并且 *不会引入新的性能问题*。**

**总结你的分析和下一步方向:**

你的分析 **再次抓住了 Folio 优化在数据块处理方面的核心挑战，并提出了 *内存映射表* 的解决方案，这是一个非常有价值的思路！**  **内存映射表可以桥接 Folio 的连续逻辑索引和 `f2fs_do_write_data_page` 函数所需的原始逻辑索引，使得 Folio 优化在数据块写回场景下成为可能。**

接下来，我们可以 **基于内存映射表的思路，设计更详细的 Folio 优化方案，并开始思考代码原型开发和验证的具体步骤**。  同时，**我们需要开始研读 F2FS 的 *其他写回路径* 的代码，分析 `f2fs_do_write_data_page` 函数在不同场景下的调用方式，评估改动影响范围，并制定全面的测试和验证计划**。  让我们继续深入探索，相信我们一定能够找到最佳的 Folio 优化方案！



