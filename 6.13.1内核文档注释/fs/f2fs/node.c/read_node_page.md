**相关函数:**
* [gc_node_segment](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/gc.c/gc_node_segment.md)
* [f2fs_ra_node_page](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/node.c/f2fs_ra_node_page.md)
* [__get_node_page](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/data.c/__get_node_page.md)
```c
/*
 * Caller should do after getting the following values.
 * 0: f2fs_put_page(page, 0)
 * LOCKED_PAGE or error: f2fs_put_page(page, 1)
 */
static int read_node_page(struct page *page, blk_opf_t op_flags)
{
	struct folio *folio = page_folio(page);
	struct f2fs_sb_info *sbi = F2FS_P_SB(page);
	struct node_info ni;
	struct f2fs_io_info fio = {
		.sbi = sbi,
		.type = NODE,
		.op = REQ_OP_READ,
		.op_flags = op_flags,
		.page = page,
		.encrypted_page = NULL,
	};
	int err;

	if (folio_test_uptodate(folio)) {
		if (!f2fs_inode_chksum_verify(sbi, page)) {
			folio_clear_uptodate(folio);
			return -EFSBADCRC;
		}
		return LOCKED_PAGE;
	}

	err = f2fs_get_node_info(sbi, folio->index, &ni, false);
	if (err)
		return err;

	/* NEW_ADDR can be seen, after cp_error drops some dirty node pages */
	if (unlikely(ni.blk_addr == NULL_ADDR || ni.blk_addr == NEW_ADDR)) {
		folio_clear_uptodate(folio);
		return -ENOENT;
	}

	fio.new_blkaddr = fio.old_blkaddr = ni.blk_addr;

	err = f2fs_submit_page_bio(&fio);

	if (!err)
		f2fs_update_iostat(sbi, NULL, FS_NODE_READ_IO, F2FS_BLKSIZE);

	return err;
}
```

*   **功能概述:** `read_node_page` 函数负责 **从存储设备读取一个节点页面**。  它会检查页面是否已 uptodate，进行校验和验证，获取节点信息，并提交 BIO 请求读取页面数据。

*   **函数注释:**  注释说明了调用者在获取返回值后应该如何处理页面引用计数：
    *   `0`:  成功返回 0，调用者应调用 `f2fs_put_page(page, 0)` 释放页面引用。
    *   `LOCKED_PAGE` 或 错误码:  返回 `LOCKED_PAGE` (宏定义，表示页面已锁定) 或 错误码，调用者应调用 `f2fs_put_page(page, 1)` 释放页面引用 (可能需要进行错误处理)。

*   **变量初始化:**
    ```c
    struct folio *folio = page_folio(page);
    struct f2fs_sb_info *sbi = F2FS_P_SB(page);
    struct node_info ni;
    struct f2fs_io_info fio = { ... };
    int err;
    ```
    *   `struct folio *folio = page_folio(page);`:  获取 `struct folio` 指针。
    *   `struct f2fs_sb_info *sbi = F2FS_P_SB(page);`:  从 `page` 中获取 `f2fs_sb_info` 指针。
    *   初始化 `f2fs_io_info` 结构体 `fio`，用于描述 IO 操作。
        *   `.type = NODE`:  IO 类型为节点 (`NODE`).
        *   `.op = REQ_OP_READ`:  IO 操作为读 (`REQ_OP_READ`).
        *   `.op_flags = op_flags`:  IO 操作标志，从函数参数 `op_flags` 传递，可以是 `REQ_RAHEAD` (预读) 或其他标志。

*   **Uptodate 状态检查和校验和验证:**
    ```c
    if (folio_test_uptodate(folio)) {
        if (!f2fs_inode_chksum_verify(sbi, page)) {
            folio_clear_uptodate(folio);
            return -EFSBADCRC;
        }
        return LOCKED_PAGE;
    }
    ```
    *   `if (folio_test_uptodate(folio)) { ... }`:  检查 folio 是否已经是 uptodate 状态。
    *   `if (!f2fs_inode_chksum_verify(sbi, page)) { ... }`:  如果 folio 是 uptodate，则 **进行校验和验证**。 `f2fs_inode_chksum_verify` 函数验证节点页面的校验和是否正确。
        *   `folio_clear_uptodate(folio);`:  如果校验和验证失败，则 **清除 folio 的 uptodate 标志**，表示页面数据无效。
        *   `return -EFSBADCRC;`:  返回 `-EFSBADCRC` 错误码，表示校验和错误。
    *   `return LOCKED_PAGE;`:  如果 folio 是 uptodate 且校验和验证通过，则 **返回 `LOCKED_PAGE`**，表示页面已锁定且数据有效。 **注意，这里并没有真正锁定页面，`LOCKED_PAGE` 只是一个宏定义，用于指示页面状态。**

*   **获取节点信息:**  `err = f2fs_get_node_info(sbi, folio->index, &ni, false);`:  调用 `f2fs_get_node_info` 函数获取节点信息 (`struct node_info`)。  `folio->index` 作为节点 ID (`nid`) 传递。

*   **无效块地址检查:**
    ```c
    /* NEW_ADDR can be seen, after cp_error drops some dirty node pages */
    if (unlikely(ni.blk_addr == NULL_ADDR || ni.blk_addr == NEW_ADDR)) {
        folio_clear_uptodate(folio);
        return -ENOENT;
    }
    ```
    *   检查从 `f2fs_get_node_info` 获取的节点块地址 `ni.blk_addr` 是否为 `NULL_ADDR` 或 `NEW_ADDR`。  `NULL_ADDR` 和 `NEW_ADDR` 是 F2FS 中表示无效或新分配地址的特殊值。
    *   如果 `ni.blk_addr` 为无效地址，则 **清除 folio 的 uptodate 标志**，并返回 `-ENOENT` 错误码，表示 "No such entry"。  这种情况可能发生在 checkpoint 错误导致脏节点页面被丢弃后。

*   **提交页面读取 BIO 请求:**
    ```c
    fio.new_blkaddr = fio.old_blkaddr = ni.blk_addr;
    err = f2fs_submit_page_bio(&fio);
    ```
    *   `fio.new_blkaddr = fio.old_blkaddr = ni.blk_addr;`:  将从 `f2fs_get_node_info` 获取的节点块地址 `ni.blk_addr` 设置为 `fio.new_blkaddr` 和 `fio.old_blkaddr`。
    *   `err = f2fs_submit_page_bio(&fio);`:  调用 `f2fs_submit_page_bio` 函数 **提交页面读取的 BIO 请求**。

*   **更新 IO 统计信息:**  `if (!err) f2fs_update_iostat(sbi, NULL, FS_NODE_READ_IO, F2FS_BLKSIZE);`:  如果 `f2fs_submit_page_bio` 没有返回错误，则更新节点读取 IO 统计信息.

*   **返回值:**  `return err;`:  返回 `f2fs_submit_page_bio` 的返回值，表示页面读取操作的结果。  `0` 表示成功，错误码表示失败。

*   **总结 `read_node_page`:**  `read_node_page` 函数负责 **从存储设备读取节点页面**。  它首先进行页面 uptodate 状态检查和校验和验证，然后获取节点信息，检查块地址有效性，并最终使用 `f2fs_submit_page_bio` **提交页面读取的 BIO 请求**。  函数内部进行了多重错误检查和处理，保证了节点页面读取操作的可靠性。
