 当然，让我们一起深入解析这段 XFS 文件系统进行预读 I/O 的代码流程，重点关注内存到磁盘块的映射以及磁盘 I/O 的提交过程。

**整体流程概览**

这段代码的核心目标是实现文件系统的预读功能。预读是指在应用程序实际请求数据之前，文件系统提前读取可能需要的数据到内存中，以提高后续数据访问的效率。`iomap_readahead` 函数是预读操作的入口点，它利用 `iomap_iter` 迭代文件范围，并使用文件系统提供的 `iomap_ops` 来获取文件块到磁盘块的映射关系，最终提交 I/O 请求读取磁盘块。<br>
**相关数据结构**
* [sturct readahead_control](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/include/linux/pagemap.h/readahead_control.md)<br>
**1. `iomap_readahead` 函数：预读的入口**

```c
void iomap_readahead(struct readahead_control *rac, const struct iomap_ops *ops)
{
	struct iomap_iter iter = {
		.inode	= rac->mapping->host,
		.pos	= readahead_pos(rac),
		.len	= readahead_length(rac),
	};
	/*
	初始化iomap_iter。其当前所在的位置被设置为预读中的起始位置(也就是用户指定的起始位置)
	其len注意表示的是迭代器剩余要处理的长度。
	注意在内核中的所有迭代器,
	*/
	struct iomap_readpage_ctx ctx = {
		.rac	= rac,
	};

	trace_iomap_readahead(rac->mapping->host, readahead_count(rac));

	while (iomap_iter(&iter, ops) > 0)
		iter.processed = iomap_readahead_iter(&iter, &ctx); // 假设存在 iomap_readahead_iter 函数
	/*迭代的后处理逻辑吧*/
	if (ctx.bio)/*如果在整个迭代结束了bio里还有累积的bio请求*/
		submit_bio(ctx.bio);/*提交它们*/
	if (ctx.cur_folio) {
		if (!ctx.cur_folio_in_bio)/*还没很深入理解*/
			folio_unlock(ctx.cur_folio);
	}
}
```

* **`struct readahead_control *rac`**:  描述了要预读的页面范围，包含了文件映射 (`rac->mapping`)、起始位置 (`readahead_pos(rac)`) 和长度 (`readahead_length(rac)`) 等信息。
* **`const struct iomap_ops *ops`**:  这是一个函数指针结构体，由具体的文件系统（这里是 XFS）提供，包含了文件系统特定的 I/O 映射操作函数。对于读取操作，我们关注的是 `ops->iomap_begin`。
* **`struct iomap_iter iter`**:  迭代器结构体，用于在文件范围内迭代，跟踪当前处理的位置 (`iter.pos`) 和剩余长度 (`iter.len`)。
* **`struct iomap_readpage_ctx ctx`**:  上下文结构体，用于在预读操作中传递信息，例如 `rac` 和用于构建 BIO 的 `ctx.bio`。
* **`while (iomap_iter(&iter, ops) > 0)`**:  循环调用 `iomap_iter` 函数，只要 `iomap_iter` 返回正值，就表示还有需要处理的文件范围。
* **`iter.processed = iomap_readahead_iter(&iter, &ctx);`**:  **关键步骤 (假设存在 `iomap_readahead_iter` 函数)**。这个函数（代码中未提供，但根据上下文推断）负责处理 `iomap_iter` 返回的每个文件范围的映射信息，并根据这些信息创建和提交实际的 I/O 请求。`iter.processed` 记录了本次迭代处理的长度。
* **`if (ctx.bio) submit_bio(ctx.bio);`**:  在循环结束后，如果 `ctx.bio` 中积累了待提交的 BIO (Block I/O) 请求，则调用 `submit_bio` 函数提交 I/O 请求到块设备层。
* **`if (ctx.cur_folio) ... folio_unlock(ctx.cur_folio);`**:  处理 `folio` (页面缓存的单位)，如果预读操作涉及到了页面缓存，这里会进行解锁操作。

**2. `iomap_iter` 函数：迭代文件范围并获取映射**

* [iomap_iter](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/iomap/iter.c/iomap_iter.md)

**3. `iomap_iter_advance` 函数：迭代器状态更新**

* [iomap_iter_advance](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/iomap/iter.c/iomap_iter_advance.md)

**4. `xfs_read_iomap_begin` 函数分析 (XFS 文件系统的 `iomap_begin` 实现):**

* [xfs_read_iomap_begin](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/xfs/xfs_iomap.c/xfs_read_iomap_begin.md)

---

**5. `xfs_bmapi_read` 函数分析:**

* [xfs_bmapi_read](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/xfs/xfs_iomap.c/xfs_bmapi_read.md)

---
**6. `xfs_bmbt_to_iomap` 函数：将 XFS 块映射信息转换为通用 `iomap` 结构体**

```c
int
xfs_bmbt_to_iomap(
	struct xfs_inode	*ip,
	struct iomap		*iomap,
	struct xfs_bmbt_irec	*imap,
	unsigned int		mapping_flags,
	u16			iomap_flags,
	u64			sequence_cookie)
{
	struct xfs_mount	*mp = ip->i_mount;
	struct xfs_buftarg	*target = xfs_inode_buftarg(ip);

	// ... (省略部分错误检查) ...

	if (imap->br_startblock == HOLESTARTBLOCK) {
		iomap->addr = IOMAP_NULL_ADDR;/*根据之前imap中存的start_block信息 如果是空洞 那iomap中的块地址就是无效的*/
		iomap->type = IOMAP_HOLE;
	} else if (imap->br_startblock == DELAYSTARTBLOCK ||
		   isnullstartblock(imap->br_startblock)) {
		iomap->addr = IOMAP_NULL_ADDR;
		iomap->type = IOMAP_DELALLOC;
	} else {
		xfs_daddr_t	daddr = xfs_fsb_to_db(ip, imap->br_startblock);/*这个是磁盘块号*/

		iomap->addr = BBTOB(daddr); // 磁盘块地址 (字节)
		if (mapping_flags & IOMAP_DAX)
			iomap->addr += target->bt_dax_part_off;

		if (imap->br_state == XFS_EXT_UNWRITTEN)
			iomap->type = IOMAP_UNWRITTEN;
		else
			iomap->type = IOMAP_MAPPED;

		// ... (RTG boundary handling) ...
	}
	iomap->offset = XFS_FSB_TO_B(mp, imap->br_startoff); // 文件偏移量 (字节)
	iomap->length = XFS_FSB_TO_B(mp, imap->br_blockcount); // 长度 (字节)
	if (mapping_flags & IOMAP_DAX)
		iomap->dax_dev = target->bt_daxdev;
	else
		iomap->bdev = target->bt_bdev; // 块设备
	iomap->flags = iomap_flags;
	// ... (dirty flag and cookie handling) ...
	iomap->folio_ops = &xfs_iomap_folio_ops;
	return 0;
}
```

* **`xfs_bmbt_to_iomap`**:  将 `xfs_bmbt_irec` 结构体 (XFS 特定的块映射信息) 转换为 `iomap` 结构体 (通用的块映射信息)。
* **`struct iomap *iomap`**:  输出参数，用于存储转换后的 `iomap` 结构体。
* **`struct xfs_bmbt_irec *imap`**:  输入参数，XFS 的块映射信息。
* **`iomap->type` 和 `iomap->addr`**:  **内存到磁盘块的映射关键所在**。
    * 根据 `imap->br_startblock` 的值，确定 `iomap->type` (例如 `IOMAP_HOLE`, `IOMAP_DELALLOC`, `IOMAP_MAPPED`, `IOMAP_UNWRITTEN`) 和 `iomap->addr` (磁盘块地址)。
    * **`xfs_daddr_t daddr = xfs_fsb_to_db(ip, imap->br_startblock);`**:  将文件系统块号 (`imap->br_startblock`) 转换为磁盘块号 (`daddr`)。
    * **`iomap->addr = BBTOB(daddr);`**:  将磁盘块号转换为字节地址，存储在 `iomap->addr` 中。`BBTOB` 宏通常是将块号转换为字节偏移量。
* **`iomap->offset = XFS_FSB_TO_B(mp, imap->br_startoff);`**:  将文件系统块偏移量 (`imap->br_startoff`) 转换为字节偏移量，存储在 `iomap->offset` 中。
* **`iomap->length = XFS_FSB_TO_B(mp, imap->br_blockcount);`**:  将文件系统块计数 (`imap->br_blockcount`) 转换为字节长度，存储在 `iomap->length` 中。
* **`iomap->bdev = target->bt_bdev;`**:  设置 `iomap` 对应的块设备。
* **`iomap->folio_ops = &xfs_iomap_folio_ops;`**:  设置 `iomap` 相关的 folio 操作函数。
**7. `static loff_t iomap_readahead_iter(const struct iomap_iter *iter, struct iomap_readpage_ctx *ctx)`**

```c
static loff_t iomap_readahead_iter(const struct iomap_iter *iter,
		struct iomap_readpage_ctx *ctx)
{
	loff_t length = iomap_length(iter);
	/*拿到迭代器所有剩余需要处理的文件字节长度*/
	loff_t done, ret;/*done 和 iter都是用文件字节类型表示的变量。done表示在这次iomap_readahead_iter调用中,目前处理的字节数。ret是循环中一次iomap_readpage_iter*/
	/*这个循环实际上已经是第二重循环了*/
	for (done = 0; done < length; done += ret) {
		if (ctx->cur_folio &&
		    offset_in_folio(ctx->cur_folio, iter->pos + done) == 0) {
			if (!ctx->cur_folio_in_bio)
				folio_unlock(ctx->cur_folio);
			ctx->cur_folio = NULL;
		}
		if (!ctx->cur_folio) {
			ctx->cur_folio = readahead_folio(ctx->rac);
			ctx->cur_folio_in_bio = false;
		}
		ret = iomap_readpage_iter(iter, ctx, done);
		if (ret <= 0)
			return ret;
	}

	return done;
}
```

* **`static loff_t iomap_readahead_iter(const struct iomap_iter *iter, struct iomap_readpage_ctx *ctx)`**:
    * `iomap_readahead_iter` 函数负责处理 `iomap_iter` 函数返回的每个文件范围的映射信息，并进行实际的页面读取操作。
    * `const struct iomap_iter *iter`: 指向 `iomap_iter` 结构体的常量指针，包含了当前迭代的文件范围和映射信息。
    * `struct iomap_readpage_ctx *ctx`: 指向 `iomap_readpage_ctx` 结构体的指针，包含了预读上下文信息。
    * `loff_t`: 函数返回类型为 `loff_t`，表示处理的长度。

* **`loff_t length = iomap_length(iter);`**:
    * 获取当前迭代需要处理的总长度，从 `iter->iomap.length` 中获取。

* **`loff_t done, ret;`**:
    * 声明两个 `loff_t` 类型的变量 `done` 和 `ret`。
        * `done`:  用于记录已经处理的长度。
        * `ret`:  用于存储每次 `iomap_readpage_iter` 函数调用的返回值。

* **`for (done = 0; done < length; done += ret)`**:
    * 这是一个 `for` 循环，用于迭代处理当前文件范围内的每个页面。
    * `done = 0`:  初始化已处理长度 `done` 为 0。
    * `done < length`:  循环条件，只要已处理长度 `done` 小于总长度 `length`，循环就继续执行。
    * `done += ret`:  每次循环结束后，将 `ret` 的值加到 `done` 上，更新已处理长度。

* **`if (ctx->cur_folio && ...)`**:
    * 条件判断：检查是否需要切换到新的 folio。
    * `ctx->cur_folio`:  检查 `ctx->cur_folio` 是否为空。如果不为空，表示当前正在使用一个 folio。
    * `offset_in_folio(ctx->cur_folio, iter->pos + done) == 0`:  检查当前处理位置 `iter->pos + done` 在当前 folio 中的偏移量是否为 0。`offset_in_folio` 宏通常用于计算偏移量在一个 folio 中的位置。如果偏移量为 0，表示当前处理位置是新 folio 的起始位置。
    * 如果条件成立（即当前有 folio 并且当前位置是新 folio 的起始位置），则需要切换到新的 folio。

* **`if (!ctx->cur_folio_in_bio) folio_unlock(ctx->cur_folio);`**:
    * 如果需要切换 folio，并且当前 folio (`ctx->cur_folio`) 没有被包含在 BIO 请求中（`!ctx->cur_folio_in_bio`），则调用 `folio_unlock(ctx->cur_folio)` 解锁当前 folio。

* **`ctx->cur_folio = NULL;`**:
    * 将 `ctx->cur_folio` 设置为 `NULL`，表示当前 folio 已经处理完毕，需要获取新的 folio。

* **`if (!ctx->cur_folio)`**:
    * 条件判断：检查 `ctx->cur_folio` 是否为空。如果为空，表示需要获取新的 folio。

* **`ctx->cur_folio = readahead_folio(ctx->rac);`**:
    * 调用 `readahead_folio(ctx->rac)` 函数，从预读控制结构体 `rac` 中获取一个新的 folio。`readahead_folio` 函数可能负责从页面缓存中分配或查找一个 folio 用于预读。

* **`ctx->cur_folio_in_bio = false;`**:
    * 将 `ctx->cur_folio_in_bio` 设置为 `false`，表示新获取的 folio 还没有被包含在 BIO 请求中。

* **`ret = iomap_readpage_iter(iter, ctx, done);`**:
    * **核心步骤：页面读取操作。** 调用 `iomap_readpage_iter` 函数进行实际的页面读取操作。
    * `iter`:  传递 `iomap_iter` 结构体的指针，包含了当前迭代的文件范围和映射信息。
    * `ctx`:  传递 `iomap_readpage_ctx` 结构体的指针，包含了预读上下文信息，例如当前的 folio。
    * `done`:  传递当前已处理的长度。
    * `iomap_readpage_iter` 函数负责根据 `iter` 中的映射信息和 `ctx` 中的 folio 信息，进行实际的页面读取操作，并将本次操作处理的长度返回。

* **`if (ret <= 0) return ret;`**:
    * 检查 `iomap_readpage_iter` 函数的返回值 `ret`。
    * 如果 `ret` 小于等于 0，表示页面读取操作失败或遇到错误，则直接返回 `ret`，终止 `iomap_readahead_iter` 函数。

* **`return done;`**:
    * 如果 `for` 循环正常结束（即处理完当前文件范围内的所有页面），则返回 `done`，表示本次 `iomap_readahead_iter` 函数处理的总长度。

**总结 `iomap_readahead_iter` 函数:**

`iomap_readahead_iter` 函数负责处理 `iomap_iter` 迭代出的每个文件范围的映射信息，并进行实际的页面读取操作。它在一个循环中迭代处理当前范围内的每个页面，管理 folio 的切换和获取，并调用 `iomap_readpage_iter` 函数进行实际的页面读取。`iomap_readahead_iter` 函数的返回值表示本次处理的长度，用于更新 `iomap_iter` 迭代器的状态。

---

**iomap 循环机制的协同工作**

这三个函数 (`iomap_readahead`, `iomap_iter`, `iomap_readahead_iter`) 协同工作，形成了一个预读 I/O 的循环机制：

1. **`iomap_readahead` 启动预读**:  `iomap_readahead` 函数作为预读的入口，初始化迭代器 `iter` 和上下文 `ctx`，并启动主循环。

2. **`iomap_iter` 迭代文件范围和获取映射**:  在主循环中，`iomap_readahead` 函数调用 `iomap_iter` 函数。`iomap_iter` 负责：
    * **迭代文件范围**:  根据迭代器的状态 (`iter->pos`, `iter->len`, `iter->processed`)，确定当前需要处理的文件范围。
    * **获取块映射信息**:  调用文件系统提供的 `ops->iomap_begin` 函数，获取当前文件范围的文件块到磁盘块的映射信息，并将映射信息存储在 `iter->iomap` 结构体中。
    * **控制循环**:  `iomap_iter` 函数的返回值决定了 `iomap_readahead` 函数的主循环是否继续进行。返回正值表示还有更多范围需要处理，返回 0 或负值表示迭代结束或发生错误。

3. **`iomap_readahead_iter` 处理映射信息和页面读取**:  对于 `iomap_iter` 返回的每个文件范围的映射信息，`iomap_readahead` 函数调用 `iomap_readahead_iter` 函数。`iomap_readahead_iter` 负责：
    * **管理 folio**:  获取和切换 folio (页面缓存单位)，用于存储读取的数据。
    * **页面读取**:  调用 `iomap_readpage_iter` 函数，根据 `iomap_iter` 提供的映射信息，进行实际的页面读取操作，将磁盘数据读取到 folio 中。
    * **返回处理长度**:  返回本次 `iomap_readahead_iter` 函数处理的长度，这个长度会被 `iomap_readahead` 函数赋值给 `iter.processed`，用于更新迭代器的状态。

4. **循环往复**:  `iomap_readahead` 函数的主循环会不断调用 `iomap_iter` 和 `iomap_readahead_iter`，直到 `iomap_iter` 返回 0 或负值，表示预读操作完成或发生错误。

5. **提交 BIO 和清理**:  在主循环结束后，`iomap_readahead` 函数会提交可能积累的 BIO 请求，并将 folio 进行清理（例如解锁）。

**总结来说，`iomap_iter` 负责迭代和获取映射，`iomap_readahead_iter` 负责处理映射和页面读取，`iomap_readahead` 协调这两个函数，形成一个完整的预读 I/O 循环机制。** 这种分工合作的设计使得预读操作可以有效地处理文件系统的块映射，并进行高效的页面读取。



**总结：内存到磁盘块的映射和 I/O 提交**

1. **内存到磁盘块的映射**:
   - `xfs_read_iomap_begin` 函数被 `iomap_iter` 调用，负责获取指定文件范围的块映射信息。
   - `xfs_read_iomap_begin` 内部调用 `xfs_bmapi_read` 函数，从 XFS inode 的 extent 树中读取块映射信息，存储在 `struct xfs_bmbt_irec imap` 结构体中。
   - `xfs_bmbt_to_iomap` 函数将 `xfs_bmbt_irec` 结构体转换为通用的 `struct iomap iomap` 结构体。
   - 在 `xfs_bmbt_to_iomap` 函数中，**关键的映射步骤**是将 `imap->br_startblock` (文件系统块号) 转换为 `iomap->addr` (磁盘块字节地址)。通过 `xfs_fsb_to_db` 和 `BBTOB` 等函数完成转换。`iomap` 结构体最终包含了文件偏移量 (`iomap->offset`)、长度 (`iomap->length`) 和对应的磁盘块地址 (`iomap->addr`)，以及块设备信息 (`iomap->bdev`)。

2. **磁盘 I/O 的提交**:
   - 在 `iomap_readahead` 函数的 `while` 循环中，**假设存在 `iomap_readahead_iter` 函数**，这个函数会根据 `iomap_iter` 返回的 `iomap` 结构体，创建 BIO (Block I/O) 请求。
   - BIO 请求会描述要读取的磁盘块范围 (通过 `iomap->addr` 和 `iomap->length` 获取) 和目标内存缓冲区 (通常是页面缓存中的页面)。
   - `iomap_readahead_iter` 函数会将多个小的 I/O 请求合并成一个大的 BIO 请求，以提高 I/O 效率。
   - 最终，在 `iomap_readahead` 函数的末尾，调用 `submit_bio(ctx.bio)` 函数将构建好的 BIO 请求提交到块设备层，由块设备驱动程序负责将 I/O 请求发送到磁盘设备执行。

**总结来说，XFS 预读 I/O 的流程是：**

1. **迭代文件范围**: `iomap_iter` 迭代预读的文件范围。
2. **获取块映射**: `xfs_read_iomap_begin` 和 `xfs_bmapi_read` 从 XFS 元数据中获取文件块到磁盘块的映射关系，并转换为 `iomap` 结构体。
3. **构建和提交 BIO**:  **假设 `iomap_readahead_iter` 函数** 根据 `iomap` 信息构建 BIO 请求，并在 `iomap_readahead` 函数中调用 `submit_bio` 提交 I/O 请求。
4. **数据读取**: 块设备驱动程序接收 BIO 请求，并驱动磁盘设备读取数据到内存 (页面缓存)。

希望这个详细的解析能够帮助你理解 XFS 预读 I/O 的流程，以及内存到磁盘块的映射和 I/O 提交过程。如果你有任何其他问题，请随时提出。

