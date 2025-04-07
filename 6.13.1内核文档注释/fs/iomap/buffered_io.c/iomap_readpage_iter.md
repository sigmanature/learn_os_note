  好的，让我们继续深入 `iomap_readpage_iter` 函数，并结合之前的预读 4 页面的例子，一步一步地分析这个函数是如何真正开始进行 I/O 操作的。

**`static loff_t iomap_readpage_iter(const struct iomap_iter *iter, struct iomap_readpage_ctx *ctx, loff_t offset)`**

```c
static loff_t iomap_readpage_iter(const struct iomap_iter *iter,
		struct iomap_readpage_ctx *ctx, loff_t offset)
{
	const struct iomap *iomap = &iter->iomap;/*先拿到iomap*/
	loff_t pos = iter->pos + offset;/*这个offset是iomap_readahead外循环的done变量
    表示已经在当前这个iomap中处理的字节量。这个offset是在[0,iomap_length(iter)]之间的*/
	loff_t length = iomap_length(iter) - offset;/*显然是指当前这个iomap还剩多少字节要处理*/
	struct folio *folio = ctx->cur_folio;
	struct iomap_folio_state *ifs;
	loff_t orig_pos = pos;
	size_t poff, plen;
	sector_t sector;

	if (iomap->type == IOMAP_INLINE)
		return iomap_read_inline_data(iter, folio);

	/* zero post-eof blocks as the page may be mapped */
	ifs = ifs_alloc(iter->inode, folio, iter->flags);
	iomap_adjust_read_range(iter->inode, folio, &pos, length, &poff, &plen);
	if (plen == 0)
		goto done;

	if (iomap_block_needs_zeroing(iter, pos)) {
		folio_zero_range(folio, poff, plen);
		iomap_set_range_uptodate(folio, poff, plen);
		goto done;
	}

	ctx->cur_folio_in_bio = true;
	if (ifs) {
		spin_lock_irq(&ifs->state_lock);
		ifs->read_bytes_pending += plen;
		spin_unlock_irq(&ifs->state_lock);
	}

	sector = iomap_sector(iomap, pos);
	if (!ctx->bio ||
	    bio_end_sector(ctx->bio) != sector ||
	    !bio_add_folio(ctx->bio, folio, plen, poff)) {
		gfp_t gfp = mapping_gfp_constraint(folio->mapping, GFP_KERNEL);
		gfp_t orig_gfp = gfp;
		unsigned int nr_vecs = DIV_ROUND_UP(length, PAGE_SIZE);

		if (ctx->bio)
			submit_bio(ctx->bio);

		if (ctx->rac) /* same as readahead_gfp_mask */
			gfp |= __GFP_NORETRY | __GFP_NOWARN;
		ctx->bio = bio_alloc(iomap->bdev, bio_max_segs(nr_vecs),
				     REQ_OP_READ, gfp);
		/*
		 * If the bio_alloc fails, try it again for a single page to
		 * avoid having to deal with partial page reads.  This emulates
		 * what do_mpage_read_folio does.
		 */
		if (!ctx->bio) {
			ctx->bio = bio_alloc(iomap->bdev, 1, REQ_OP_READ,
					     orig_gfp);
		}
		if (ctx->rac)
			ctx->bio->bi_opf |= REQ_RAHEAD;
		ctx->bio->bi_iter.bi_sector = sector;
		ctx->bio->bi_end_io = iomap_read_end_io;
		bio_add_folio_nofail(ctx->bio, folio, plen, poff);
	}

done:
	/*
	 * Move the caller beyond our range so that it keeps making progress.
	 * For that, we have to include any leading non-uptodate ranges, but
	 * we can skip trailing ones as they will be handled in the next
	 * iteration.
	 */
	return pos - orig_pos + plen;
}
```

* **`static loff_t iomap_readpage_iter(const struct iomap_iter *iter, struct iomap_readpage_ctx *ctx, loff_t offset)`**:
    * 函数签名，`iomap_readpage_iter` 负责实际的页面读取操作。
    * `const struct iomap_iter *iter`:  指向 `iomap_iter` 结构体的指针，包含当前迭代的映射信息 (`iter->iomap`) 和文件位置 (`iter->pos`).
    * `struct iomap_readpage_ctx *ctx`: 指向 `iomap_readpage_ctx` 结构体的指针，包含预读上下文，例如当前的 `folio` (`ctx->cur_folio`) 和 `bio` (`ctx->bio`).
    * `loff_t offset`:  当前页面在 `iter->iomap` 映射范围内的偏移量。

* **`const struct iomap *iomap = &iter->iomap;`**:
    * 获取当前迭代的映射信息，方便后续使用。

* **`loff_t pos = iter->pos + offset;`**:
    * 计算当前页面在文件中的绝对偏移量 `pos`。`iter->pos` 是 `iomap_iter` 当前迭代的起始文件偏移量，`offset` 是在当前映射范围内的偏移。

* **`loff_t length = iomap_length(iter) - offset;`**:
    * 计算当前页面需要读取的长度 `length`。`iomap_length(iter)` 是当前映射的总长度，减去 `offset` 得到剩余需要读取的长度。

* **`struct folio *folio = ctx->cur_folio;`**:
    * 从上下文 `ctx` 中获取当前正在使用的 `folio`。`folio` 是页面缓存的单位，用于存储读取的数据。

* **`struct iomap_folio_state *ifs;`**:
    * 声明一个指向 `iomap_folio_state` 结构体的指针 `ifs`。`iomap_folio_state` 用于跟踪每个 `folio` 的块状态（uptodate, dirty 等）。

* **`loff_t orig_pos = pos;`**:
    * 保存原始的 `pos` 值，用于后续计算返回值。

* **`size_t poff, plen;`**:
    * 声明 `poff` 和 `plen` 变量，用于存储页面内偏移量 (page offset) 和页面内长度 (page length)。

* **`sector_t sector;`**:
    * 声明 `sector` 变量，用于存储起始扇区号。

* **`if (iomap->type == IOMAP_INLINE) return iomap_read_inline_data(iter, folio);`**:
    * **Inline 数据处理 (跳过):** 如果 `iomap->type` 是 `IOMAP_INLINE`，表示数据是内联存储在 inode 中的，调用 `iomap_read_inline_data` 函数处理。  **在我们的例子中，我们假设是非内联数据，所以会跳过这个 `if` 语句。**

* **`ifs = ifs_alloc(iter->inode, folio, iter->flags);`**:
    * **分配 `iomap_folio_state`**: 调用 `ifs_alloc` 函数为当前的 `folio` 分配 `iomap_folio_state` 结构体，并将 `ifs` 指针指向它。
    * `ifs_alloc` 函数可能负责分配内存并初始化 `iomap_folio_state` 结构体，用于跟踪 `folio` 中每个块的状态。

* **`iomap_adjust_read_range(iter->inode, folio, &pos, length, &poff, &plen);`**:
    * **调整读取范围**: 调用 `iomap_adjust_read_range` 函数，根据 `folio` 的状态和 inode 的信息，调整实际需要读取的范围。
    * `iter->inode`:  文件 inode。
    * `folio`:  当前 `folio`。
    * `&pos`:  指向当前文件偏移量 `pos` 的指针 (输入/输出参数)。`iomap_adjust_read_range` 可能会修改 `pos`，例如跳过已经 uptodate 的块。
    * `length`:  请求读取的总长度 (输入参数)。
    * `&poff`:  指向页面内偏移量 `poff` 的指针 (输出参数)。`iomap_adjust_read_range` 会设置 `poff` 为实际读取数据在 `folio` 中的起始偏移量。
    * `&plen`:  指向页面内长度 `plen` 的指针 (输出参数)。`iomap_adjust_read_range` 会设置 `plen` 为实际需要读取的长度。
    * **`iomap_adjust_read_range` 的作用**:  优化读取操作，避免重复读取已经 uptodate 的数据块，并处理文件大小 `i_size` 边界的情况。

* **`if (plen == 0) goto done;`**:
    * **检查是否需要读取**: 如果 `iomap_adjust_read_range` 函数调整后的 `plen` 为 0，表示当前 `folio` 的目标范围已经全部 uptodate，不需要实际读取数据，直接跳转到 `done` 标签。

* **`if (iomap_block_needs_zeroing(iter, pos)) { ... }`**:
    * **检查是否需要零填充**: 调用 `iomap_block_needs_zeroing` 函数，检查是否需要对当前范围进行零填充。
    * `iomap_block_needs_zeroing` 函数可能根据 `iomap->type` 和文件系统状态判断是否需要零填充，例如对于 hole 或 DELALLOC 类型的映射，可能需要零填充。
    * **零填充操作**: 如果 `iomap_block_needs_zeroing` 返回真，则执行以下操作：
        * `folio_zero_range(folio, poff, plen);`: 调用 `folio_zero_range` 函数，将 `folio` 中从偏移量 `poff` 开始，长度为 `plen` 的范围零填充。
        * `iomap_set_range_uptodate(folio, poff, plen);`: 调用 `iomap_set_range_uptodate` 函数，将 `folio` 中零填充的范围标记为 uptodate。
        * `goto done;`: 跳转到 `done` 标签，完成当前页面的处理。
    * **在我们的预读例子中，假设映射类型是 `IOMAP_MAPPED`，并且不需要零填充，所以会跳过这个 `if` 语句。**

* **`ctx->cur_folio_in_bio = true;`**:
    * **标记 folio 已加入 BIO**: 设置 `ctx->cur_folio_in_bio` 为 `true`，表示当前的 `folio` 已经被加入到 BIO 请求中，即将进行 I/O 操作。

* **`if (ifs) { ... }`**:
    * **更新 `iomap_folio_state`**: 如果 `ifs` 指针不为空（即成功分配了 `iomap_folio_state` 结构体），则执行以下操作：
        * `spin_lock_irq(&ifs->state_lock);`: 获取 `ifs->state_lock` 自旋锁，保护 `iomap_folio_state` 结构体的并发访问。
        * `ifs->read_bytes_pending += plen;`:  将实际要读取的长度 `plen` 加到 `ifs->read_bytes_pending` 计数器上，记录待完成的读取字节数。
        * `spin_unlock_irq(&ifs->state_lock);`: 释放 `ifs->state_lock` 自旋锁。

* **`sector = iomap_sector(iomap, pos);`**:
    * **计算起始扇区号**: 调用 `iomap_sector` 函数，根据 `iomap` 结构体和当前文件偏移量 `pos`，计算出要读取的磁盘扇区号 `sector`。
    * `iomap_sector` 函数会将文件偏移量转换为相对于磁盘起始位置的扇区号。

* **`if (!ctx->bio || ...)`**:
    * **判断是否需要创建新的 BIO**:  判断是否需要创建新的 BIO 请求。以下条件之一满足时，就需要创建新的 BIO：
        * `!ctx->bio`:  `ctx->bio` 为空，表示当前还没有 BIO 请求。
        * `bio_end_sector(ctx->bio) != sector`:  当前 `ctx->bio` 请求的结束扇区号不等于当前要读取的起始扇区号 `sector`，表示当前的 BIO 请求无法覆盖当前要读取的范围，需要创建新的 BIO。
        * `!bio_add_folio(ctx->bio, folio, plen, poff)`:  尝试将当前的 `folio` 添加到 `ctx->bio` 中，如果 `bio_add_folio` 返回失败（例如，BIO 请求已满或无法合并），则需要创建新的 BIO。

* **BIO 创建和提交 (如果需要创建新 BIO)**: 如果上述 `if` 条件成立，则执行以下操作：
    * **`gfp_t gfp = mapping_gfp_constraint(folio->mapping, GFP_KERNEL);`**: 获取内存分配标志 `gfp`，用于 BIO 分配。`mapping_gfp_constraint` 函数可能根据 `folio->mapping` 的属性返回合适的 `gfp` 标志。
    * **`gfp_t orig_gfp = gfp;`**: 保存原始的 `gfp` 标志。
    * **`unsigned int nr_vecs = DIV_ROUND_UP(length, PAGE_SIZE);`**: 计算需要的 BIO segment 数量 `nr_vecs`，用于分配 BIO 结构体。
    * **`if (ctx->bio) submit_bio(ctx->bio);`**: 如果 `ctx->bio` 不为空（表示之前有积累的 BIO 请求），则先调用 `submit_bio(ctx->bio)` 提交之前的 BIO 请求。
    * **`if (ctx->rac) gfp |= __GFP_NORETRY | __GFP_NOWARN;`**: 如果存在预读控制结构体 `ctx->rac`，则在 `gfp` 标志中添加 `__GFP_NORETRY` 和 `__GFP_NOWARN`，用于预读操作的内存分配，表示分配失败时不需要重试和警告。
    * **`ctx->bio = bio_alloc(iomap->bdev, bio_max_segs(nr_vecs), REQ_OP_READ, gfp);`**: 调用 `bio_alloc` 函数分配一个新的 BIO 结构体，并赋值给 `ctx->bio`。
        * `iomap->bdev`:  块设备。
        * `bio_max_segs(nr_vecs)`:  BIO 最大 segment 数量。
        * `REQ_OP_READ`:  BIO 操作类型为读取。
        * `gfp`:  内存分配标志。
    * **BIO 分配失败重试 (单页 BIO)**:
        ```c
        if (!ctx->bio) {
            ctx->bio = bio_alloc(iomap->bdev, 1, REQ_OP_READ, orig_gfp);
        }
        ```
        如果 `bio_alloc` 分配 BIO 失败，则尝试再次分配一个单页 BIO (segment 数量为 1)，使用原始的 `orig_gfp` 标志。这是为了避免处理部分页面读取的复杂性，类似于 `do_mpage_read_folio` 的处理方式。
    * **`if (ctx->rac) ctx->bio->bi_opf |= REQ_RAHEAD;`**: 如果存在预读控制结构体 `ctx->rac`，则设置 BIO 的操作标志 `bi_opf` 包含 `REQ_RAHEAD`，表示这是一个预读请求。
    * **`ctx->bio->bi_iter.bi_sector = sector;`**: 设置 BIO 的起始扇区号 `bi_sector` 为之前计算的 `sector`。
    * **`ctx->bio->bi_end_io = iomap_read_end_io;`**: 设置 BIO 的完成回调函数 `bi_end_io` 为 `iomap_read_end_io`。当 BIO 完成时，会调用 `iomap_read_end_io` 函数进行后续处理。
    * **`bio_add_folio_nofail(ctx->bio, folio, plen, poff);`**: 调用 `bio_add_folio_nofail` 函数将当前的 `folio` 添加到新创建的 `ctx->bio` 请求中。
        * `ctx->bio`:  新创建的 BIO 请求。
        * `folio`:  当前 `folio`。
        * `plen`:  要读取的长度。
        * `poff`:  页面内偏移量。
        * `bio_add_folio_nofail` 函数将 `folio` 映射到 BIO 的 segment 中。

* **`else { bio_add_folio(ctx->bio, folio, plen, poff); }`**:
    * **BIO 扩展 (如果不需要创建新 BIO)**: 如果 `if (!ctx->bio || ...)` 条件不成立，表示可以使用已有的 `ctx->bio` 请求，则调用 `bio_add_folio` 函数将当前的 `folio` 添加到已有的 `ctx->bio` 请求中。
    * `bio_add_folio` 函数尝试将 `folio` 合并到现有的 BIO 请求中。如果合并成功，则继续使用现有的 BIO。

* **`done:`**:  标签，用于跳转到这里。

* **`return pos - orig_pos + plen;`**:
    * **返回值**: 返回本次 `iomap_readpage_iter` 函数处理的长度。
    * `pos - orig_pos`:  由于 `iomap_adjust_read_range` 函数可能会修改 `pos`，`pos - orig_pos` 计算了由于跳过 uptodate 块而导致的 `pos` 的变化量。
    * `plen`:  实际读取的长度。
    * `pos - orig_pos + plen`:  最终返回的值表示本次 `iomap_readpage_iter` 函数处理的总长度，包括跳过的 uptodate 块和实际读取的块。这个返回值会被 `iomap_readahead_iter` 函数用于更新迭代进度。

**`static inline sector_t iomap_sector(const struct iomap *iomap, loff_t pos)`**

```c
static inline sector_t iomap_sector(const struct iomap *iomap, loff_t pos)
{
	return (iomap->addr + pos - iomap->offset) >> SECTOR_SHIFT;
}
```

* **`static inline sector_t iomap_sector(const struct iomap *iomap, loff_t pos)`**:
    * 内联函数 `iomap_sector`，用于将文件偏移量转换为磁盘扇区号。
    * `const struct iomap *iomap`: 指向 `iomap` 结构体的指针，包含磁盘地址 (`iomap->addr`) 和文件偏移量 (`iomap->offset`) 信息。
    * `loff_t pos`:  当前文件偏移量。
    * **计算公式**: `(iomap->addr + pos - iomap->offset) >> SECTOR_SHIFT`
        * `iomap->addr`:  `iomap` 结构体中存储的磁盘块的起始字节地址。
        * `pos - iomap->offset`:  计算当前文件偏移量 `pos` 相对于 `iomap` 映射起始文件偏移量 `iomap->offset` 的偏移量。
        * `iomap->addr + (pos - iomap->offset)`:  计算当前文件偏移量 `pos` 对应的磁盘字节地址。
        * `>> SECTOR_SHIFT`:  将磁盘字节地址右移 `SECTOR_SHIFT` 位，转换为扇区号。`SECTOR_SHIFT` 通常为 9 (扇区大小为 512 字节)。

**`static void iomap_adjust_read_range(struct inode *inode, struct folio *folio, loff_t *pos, loff_t length, size_t *offp, size_t *lenp)`**

```c
static void iomap_adjust_read_range(struct inode *inode, struct folio *folio,
		loff_t *pos, loff_t length, size_t *offp, size_t *lenp)
{
	// ... (代码细节，主要涉及块大小、uptodate 状态检查和 i_size 处理) ...
}
```

* **`static void iomap_adjust_read_range(...)`**:
    * 函数 `iomap_adjust_read_range` 负责调整实际需要读取的范围，以优化读取操作。
    * **主要功能**:
        * **跳过 uptodate 块**: 检查 `folio` 中每个块的 uptodate 状态，跳过已经 uptodate 的块，只读取需要读取的块。
        * **处理 `i_size` 边界**:  如果读取范围跨越了文件大小 `i_size` 的边界，需要进行特殊处理，避免读取超出 `i_size` 范围的数据，并确保超出 `i_size` 范围的部分被零填充。
        * **块大小适配**:  如果文件系统的块大小小于页面大小，需要按块大小进行更精细的 uptodate 状态检查和范围调整。

**`struct iomap_folio_state { ... }`**

```c
struct iomap_folio_state {
	spinlock_t		state_lock;
	unsigned int		read_bytes_pending;
	atomic_t		write_bytes_pending;
	unsigned long		state[]; // Bitmap for uptodate and dirty status
};
```

* **`struct iomap_folio_state`**:
    * 结构体 `iomap_folio_state` 用于跟踪每个 `folio` 的状态信息。
    * `spinlock_t state_lock`:  自旋锁，用于保护结构体的并发访问。
    * `unsigned int read_bytes_pending`:  记录待完成的读取字节数。
    * `atomic_t write_bytes_pending`:  原子计数器，记录待完成的写入字节数。
    * `unsigned long state[]`:  位图，用于存储 `folio` 中每个块的 uptodate 和 dirty 状态。每个块占用两位，一位表示 uptodate 状态，一位表示 dirty 状态。

**继续我们的例子**

让我们回到之前的例子，假设我们正在处理第一次调用 `iomap_readahead_iter`，并且 `iter->iomap` 包含了文件偏移量 0 ~ 2 * `PAGE_SIZE` (8KB) 的映射信息，磁盘地址为 `0x1000`。

1. **`pos = iter->pos + offset;`**:  假设 `offset = 0`，则 `pos = 0 + 0 = 0`。
2. **`length = iomap_length(iter) - offset;`**: `length = 2 * PAGE_SIZE - 0 = 2 * PAGE_SIZE`。
3. **`ifs = ifs_alloc(...)`**:  分配 `iomap_folio_state` 结构体。
4. **`iomap_adjust_read_range(...)`**:  假设 `folio` 中对应的范围都不是 uptodate，`iomap_adjust_read_range` 不会修改 `pos` 和 `length`，`poff` 会设置为 0，`plen` 会设置为 `PAGE_SIZE` (假设 `iomap_readpage_iter` 一次处理一个页面)。
5. **`if (plen == 0) goto done;`**: `plen` 为 `PAGE_SIZE`，条件不成立。
6. **`if (iomap_block_needs_zeroing(...))`**:  假设不需要零填充，条件不成立。
7. **`ctx->cur_folio_in_bio = true;`**:  标记 `folio` 已加入 BIO。
8. **`if (ifs) { ... }`**: 更新 `ifs->read_bytes_pending`。
9. **`sector = iomap_sector(iomap, pos);`**: 计算起始扇区号，`sector = (0x1000 + 0 - 0) >> SECTOR_SHIFT`。
10. **`if (!ctx->bio || ...)`**:  假设 `ctx->bio` 为空，条件成立，需要创建新的 BIO。
11. **BIO 创建**:  分配新的 BIO 结构体 `ctx->bio`，设置 `bi_sector = sector`, `bi_end_io = iomap_read_end_io`, `bi_opf |= REQ_RAHEAD`。
12. **`bio_add_folio_nofail(ctx->bio, folio, plen, poff);`**: 将 `folio` 添加到 `ctx->bio` 中，读取长度为 `plen = PAGE_SIZE`，页面内偏移量为 `poff = 0`。

在第一次 `iomap_readpage_iter` 调用中，会创建一个 BIO 请求，读取第一个页面 (4KB) 的数据。在后续的 `iomap_readpage_iter` 调用中，如果读取范围仍然可以合并到同一个 BIO 中，则会继续扩展 `ctx->bio`，否则可能会创建新的 BIO 请求。最终，当所有页面都添加到 BIO 后，会在 `iomap_readahead` 函数的末尾调用 `submit_bio(ctx->bio)` 提交 BIO 请求，触发实际的磁盘 I/O 操作。

希望这个详细的分析能够帮助你理解 `iomap_readpage_iter` 函数的内部逻辑，以及它是如何与 `folio` 和 `bio` 交互，最终发起磁盘 I/O 请求的。

