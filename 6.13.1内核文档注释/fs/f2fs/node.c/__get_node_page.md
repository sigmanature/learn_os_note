**相关函数:**
* [f2fs_ra_node_page](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/node.c/f2fs_ra_node_page.md)
* [read_node_page](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/node.c/read_node_page.md)
```c
static struct page *__get_node_page(struct f2fs_sb_info *sbi, pgoff_t nid,
					struct page *parent, int start)
{
	struct page *page;
	int err;

	if (!nid)
		return ERR_PTR(-ENOENT);
	if (f2fs_check_nid_range(sbi, nid))
		return ERR_PTR(-EINVAL);
repeat:
	page = f2fs_grab_cache_page(NODE_MAPPING(sbi), nid, false);
	if (!page)
		return ERR_PTR(-ENOMEM);

	err = read_node_page(page, 0);
	if (err < 0) {
		goto out_put_err;
	} else if (err == LOCKED_PAGE) {
		err = 0;
		goto page_hit;
	}

	if (parent)
		f2fs_ra_node_pages(parent, start + 1, MAX_RA_NODE);

	lock_page(page);

	if (unlikely(page->mapping != NODE_MAPPING(sbi))) {
		f2fs_put_page(page, 1);
		goto repeat;
	}

	if (unlikely(!PageUptodate(page))) {
		err = -EIO;
		goto out_err;
	}

	if (!f2fs_inode_chksum_verify(sbi, page)) {
		err = -EFSBADCRC;
		goto out_err;
	}
	page_hit:
	if (likely(nid == nid_of_node(page)))
		return page;

	f2fs_warn(sbi, "inconsistent node block, nid:%lu, node_footer[nid:%u,ino:%u,ofs:%u,cpver:%llu,blkaddr:%u]",
			  nid, nid_of_node(page), ino_of_node(page),
			  ofs_of_node(page), cpver_of_node(page),
			  next_blkaddr_of_node(page));
	set_sbi_flag(sbi, SBI_NEED_FSCK);
	f2fs_handle_error(sbi, ERROR_INCONSISTENT_FOOTER);
	err = -EFSCORRUPTED;
out_err:
	ClearPageUptodate(page);
out_put_err:
	/* ENOENT comes from read_node_page which is not an error. */
	if (err != -ENOENT)
		f2fs_handle_page_eio(sbi, page_folio(page), NODE);
	f2fs_put_page(page, 1);
	return ERR_PTR(err);
}
```

*   **功能:** `__get_node_page` 函数是 **获取节点页面的核心函数**。  `f2fs_get_node_page` 函数只是 `__get_node_page` 的一个简单封装。  `__get_node_page` 函数负责从页面缓存或磁盘读取节点页面，并进行一系列的错误检查和处理。

*   **参数:**
    *   `struct f2fs_sb_info *sbi`: 文件系统超级块信息。
    *   `pgoff_t nid`:  要获取的节点页面的 NID。
    *   `struct page *parent`:  父页面指针，可能用于预读优化 (在 `f2fs_ra_node_pages` 中使用)。
    *   `int start`:  起始偏移量，可能用于预读优化 (在 `f2fs_ra_node_pages` 中使用)。

*   **无效 NID 检查:**  函数开始处进行无效 NID 检查，如果 NID 无效，则返回错误。

*   **`repeat:` 标签和页面获取循环:**
    ```c
	repeat:
	page = f2fs_grab_cache_page(NODE_MAPPING(sbi), nid, false);
	if (!page)
		return ERR_PTR(-ENOMEM);

	err = read_node_page(page, 0);
	if (err < 0) {
		goto out_put_err;
	} else if (err == LOCKED_PAGE) {
		err = 0;
		goto page_hit;
	}

	// ... (预读, 页面锁定, 错误检查) ...
    ```
    *   **`repeat:` 标签:**  用于在特定情况下重试页面获取流程。
    *   **获取页面缓存页:**  `page = f2fs_grab_cache_page(NODE_MAPPING(sbi), nid, false);`:  调用 `f2fs_grab_cache_page` 函数 **尝试从节点页面缓存中获取 NID 对应的页面**。
    *   **内存分配失败处理:**  `if (!page) return ERR_PTR(-ENOMEM);`:  如果 `f2fs_grab_cache_page` 返回 `NULL` (页面获取失败，通常是内存分配失败)，则返回 `-ENOMEM` 错误。
    *   **读取节点页面:**  `err = read_node_page(page, 0);`:  调用 `read_node_page` 函数 **从磁盘读取节点页面数据**。  `0` 表示非预读操作。
    *   **`read_node_page` 错误处理:**
        *   `if (err < 0) { goto out_put_err; }`:  如果 `read_node_page` 返回负错误码，则跳转到 `out_put_err` 标签，进行错误处理。
        *   `else if (err == LOCKED_PAGE) { err = 0; goto page_hit; }`:  如果 `read_node_page` 返回 `LOCKED_PAGE` (表示页面已 uptodate 且锁定)，则将 `err` 设置为 0，并跳转到 `page_hit` 标签，进行后续的页面命中处理。

    *   **预读优化 (条件性):**
        ```c
        if (parent)
            f2fs_ra_node_pages(parent, start + 1, MAX_RA_NODE);
        ```
        *   `if (parent) f2fs_ra_node_pages(parent, start + 1, MAX_RA_NODE);`:  如果 `parent` 参数非空 (表示有父页面)，则调用 `f2fs_ra_node_pages` 函数 **预读 *一批* 相关的节点页面**。  这是一种 **树状结构的预读优化**，例如，在读取一个 inode 页面后，可以预读其子节点的页面。

    *   **页面锁定和一致性检查:**
        ```c
        lock_page(page);

        if (unlikely(page->mapping != NODE_MAPPING(sbi))) {
            f2fs_put_page(page, 1);
            goto repeat;
        }

        if (unlikely(!PageUptodate(page))) {
            err = -EIO;
            goto out_err;
        }

        if (!f2fs_inode_chksum_verify(sbi, page)) {
            err = -EFSBADCRC;
            goto out_err;
        }
        ```
        *   **锁定页面:**  `lock_page(page);`:  **锁定页面**，防止并发访问和修改。
        *   **`mapping` 一致性检查:**  `if (unlikely(page->mapping != NODE_MAPPING(sbi))) { ... }`:  **检查页面的 `mapping` 是否仍然是预期的节点页面 mapping (`NODE_MAPPING(sbi)`)**。  如果 `mapping` 不一致，则释放页面并跳转到 `repeat` 标签，重新开始页面获取流程。  **防止页面被错误地添加到其他 mapping。**
        *   **`PageUptodate` 状态检查:**  `if (unlikely(!PageUptodate(page))) { ... }`:  **检查页面是否是 uptodate 状态**。  如果不是 uptodate，则表示页面读取失败，设置错误码为 `-EIO` (Input/Output error)，并跳转到 `out_err` 标签，进行错误处理。
        *   **校验和验证:**  `if (!f2fs_inode_chksum_verify(sbi, page)) { ... }`:  **进行节点页面校验和验证**。  如果校验和验证失败，则设置错误码为 `-EFSBADCRC` (Bad CRC)，并跳转到 `out_err` 标签，进行错误处理。

    *   **`page_hit:` 标签和 NID 一致性检查:**
	```c
	page_hit:
        if (likely(nid == nid_of_node(page)))
            return page;

        f2fs_warn(sbi, "inconsistent node block, nid:%lu, node_footer[nid:%u,ino:%u,ofs:%u,cpver:%llu,blkaddr:%u]",
                  nid, nid_of_node(page), ino_of_node(page),
                  ofs_of_node(page), cpver_of_node(page),
                  next_blkaddr_of_node(page));
        set_sbi_flag(sbi, SBI_NEED_FSCK);
        f2fs_handle_error(sbi, ERROR_INCONSISTENT_FOOTER);
        err = -EFSCORRUPTED;
 	out_err:
        ClearPageUptodate(page);
 	out_put_err:
        /* ENOENT comes from read_node_page which is not an error. */
        if (err != -ENOENT)
            f2fs_handle_page_eio(sbi, page_folio(page), NODE);
        f2fs_put_page(page, 1);
        return ERR_PTR(err);
	```
  	*   **`page_hit:` 标签:**  `read_node_page` 返回 `LOCKED_PAGE` 或页面读取成功后，跳转到 `page_hit` 标签。
        *   **NID 一致性检查:**  `if (likely(nid == nid_of_node(page))) return page;`:  **检查页面的实际 NID (`nid_of_node(page)`) 是否与请求的 NID (`nid`) 一致**。  `nid_of_node(page)` 从页面 Footer 中提取 NID 信息。  **这是一个重要的 *双重验证* 机制，确保获取到的页面确实是请求的 NID 对应的页面，防止 Page Cache 出现错误或数据损坏。**  在正常情况下，NID 应该是一致的。
        *   **不一致错误处理:**  如果 NID 不一致，则打印警告信息，设置文件系统错误标志，并返回 `-EFSCORRUPTED` 错误。
        *   **`out_err:` 标签:**  页面 uptodate 状态检查或校验和验证失败后，跳转到 `out_err` 标签。  `ClearPageUptodate(page);` 清除页面的 uptodate 标志，表示页面数据无效。
        *   **`out_put_err:` 标签:**  `read_node_page` 返回负错误码或 `out_err` 标签跳转到 `out_put_err` 标签。  `if (err != -ENOENT) f2fs_handle_page_eio(sbi, page_folio(page), NODE);` 如果错误不是 `-ENOENT` (No such entry，通常不是致命错误)，则调用 `f2fs_handle_page_eio` 函数处理页面 EIO 错误 (例如，记录日志，标记文件系统错误状态)。  `f2fs_put_page(page, 1);` 释放页面引用计数。  `return ERR_PTR(err);` 返回错误指针。
