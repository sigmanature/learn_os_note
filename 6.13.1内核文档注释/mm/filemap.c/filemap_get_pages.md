 **相关函数**
 *	[filemap_read](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/filemap.c/filemap_read.md)
 *	[filemap_get_read_batch](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/mm/filemap.c/filemap_get_read_batch.md)
 好的，非常棒！找到了 `filemap_get_pages` 和 `pos_same_folio` 的代码，这正是我们深入理解 `filemap_read` 如何获取页面的关键。让我们逐步解析这些代码。

**1. `pos_same_folio` 函数解析**

```c
static inline bool pos_same_folio(loff_t pos1, loff_t pos2, struct folio *folio)
{
	unsigned int shift = folio_shift(folio);

	return (pos1 >> shift == pos2 >> shift);
}
```

* **`static inline bool pos_same_folio(loff_t pos1, loff_t pos2, struct folio *folio)`**: 函数定义。
    * `static inline`:  `static` 表示该函数只在当前文件内可见，`inline` 建议编译器内联展开该函数，以提高性能，因为这个函数非常简单且频繁调用。
    * `bool`: 返回值类型，表示两个位置是否在同一个 folio 内。
    * `loff_t pos1, loff_t pos2`:  两个文件偏移量 (positions)。
    * `struct folio *folio`:  指向 folio 结构体的指针。
* **`unsigned int shift = folio_shift(folio);`**: 调用 `folio_shift` 函数获取 folio 的 shift 值。这个 shift 值实际上是 folio 大小的以 2 为底的对数，即 $log_2(\text{folio\_size})$。例如，如果 folio 大小是 4KB (PAGE_SIZE)，则 shift 值为 12 (因为 $2^{12} = 4096$)。如果 folio 大小是更大的，比如 16KB，shift 值会更大。
* **`return (pos1 >> shift == pos2 >> shift);`**:  核心逻辑。
    * `pos1 >> shift`:  将偏移量 `pos1` 右移 `shift` 位。这相当于将 `pos1` 除以 folio 的大小，并向下取整。结果是 `pos1` 所在的 folio 的索引 (以 folio 大小为单位)。
    * `pos2 >> shift`:  同样，计算 `pos2` 所在的 folio 的索引。
    * `==`:  比较两个索引是否相等。如果相等，则表示 `pos1` 和 `pos2` 在同一个 folio 内，函数返回 `true`，否则返回 `false`。

**总结 `pos_same_folio`:**

`pos_same_folio` 函数用于快速判断两个文件偏移量 `pos1` 和 `pos2` 是否位于同一个 folio 内。它通过将偏移量右移 folio 的 shift 值，得到 folio 的索引，然后比较索引是否相等来实现。这是一个非常高效的判断方法。

**2. `folio_shift` 函数解析**

```c
static inline unsigned int folio_shift(const struct folio *folio)
{
	return PAGE_SHIFT + folio_order(folio);
}
```

* **`static inline unsigned int folio_shift(const struct folio *folio)`**: 函数定义。
    * `static inline`:  同样是静态内联函数，原因同上。
    * `unsigned int`: 返回值类型，folio 的 shift 值。
    * `const struct folio *folio`: 指向 folio 结构体的常量指针。
* **`return PAGE_SHIFT + folio_order(folio);`**:  函数体。
    * `PAGE_SHIFT`:  宏，定义了 PAGE_SIZE 的以 2 为底的对数。对于 4KB 页，`PAGE_SHIFT` 通常是 12。
    * `folio_order(folio)`:  函数调用，返回 folio 的 order。folio 的 order 表示 folio 是由多少个连续的物理页组成的，以 2 的幂次方表示。order 0 表示 1 个页 (PAGE_SIZE)，order 1 表示 2 个页 (2 * PAGE_SIZE)，order 2 表示 4 个页 (4 * PAGE_SIZE)，以此类推。
    * `PAGE_SHIFT + folio_order(folio)`:  将 `PAGE_SHIFT` 和 `folio_order(folio)` 相加，得到 folio 的 shift 值。这实际上计算了 $log_2(\text{folio\_size}) = log_2(PAGE\_SIZE \times 2^{\text{folio\_order}})$.

**总结 `folio_shift`:**

`folio_shift` 函数计算并返回给定 folio 的 shift 值，这个值是 folio 大小的以 2 为底的对数。它通过将 `PAGE_SHIFT` (页大小的 shift 值) 加上 `folio_order(folio)` (folio 的 order) 来实现。

**3. `filemap_get_pages` 函数解析**

现在我们来详细解析 `filemap_get_pages` 函数，这是理解页面获取和预读的关键。

```c
static int filemap_get_pages(struct kiocb *iocb, size_t count,
		struct folio_batch *fbatch, bool need_uptodate)
{
	struct file *filp = iocb->ki_filp;
	struct address_space *mapping = filp->f_mapping;
	struct file_ra_state *ra = &filp->f_ra;
	pgoff_t index = iocb->ki_pos >> PAGE_SHIFT;
	pgoff_t last_index;
	struct folio *folio;
	unsigned int flags;
	int err = 0;
```

* **`static int filemap_get_pages(struct kiocb *iocb, size_t count, struct folio_batch *fbatch, bool need_uptodate)`**: 函数定义。
    * `static int`:  静态函数，返回 `int` 类型的错误码 (0 表示成功，负值表示错误)。
    * `struct kiocb *iocb`: I/O 控制块。
    * `size_t count`:  需要读取的字节数。
    * `struct folio_batch *fbatch`:  用于存储获取到的 folio 的批量结构体。
    * `bool need_uptodate`:  布尔值，指示获取的 folio 是否必须是 `Uptodate` 状态。`true` 表示必须是 `Uptodate`，`false` 表示可以先返回，稍后更新。
* **`struct file *filp = iocb->ki_filp;`**: 获取 `file` 结构体指针。
* **`struct address_space *mapping = filp->f_mapping;`**: 获取地址空间 `address_space`。
* **`struct file_ra_state *ra = &filp->f_ra;`**: 获取预读状态 `file_ra_state`。
* **`pgoff_t index = iocb->ki_pos >> PAGE_SHIFT;`**: 计算起始页索引 `index`。将读取起始位置 `iocb->ki_pos` 右移 `PAGE_SHIFT` 位，得到以页为单位的索引。`pgoff_t` 是页偏移量类型。
* **`pgoff_t last_index;`**: 声明变量 `last_index`，用于存储读取结束页的索引。
* **`struct folio *folio;`**: 声明 `folio` 指针。
* **`unsigned int flags;`**: 声明 `flags` 变量，用于保存内存分配标志。
* **`int err = 0;`**: 初始化错误码 `err` 为 0。

```c
	/* "last_index" is the index of the page beyond the end of the read */
	last_index = DIV_ROUND_UP(iocb->ki_pos + count, PAGE_SIZE);
retry:
	if (fatal_signal_pending(current))
		return -EINTR;

	filemap_get_read_batch(mapping, index, last_index - 1, fbatch);
	if (!folio_batch_count(fbatch)) {
		if (iocb->ki_flags & IOCB_NOIO)
			return -EAGAIN;
		if (iocb->ki_flags & IOCB_NOWAIT)
			flags = memalloc_noio_save();
		page_cache_sync_readahead(mapping, ra, filp, index,
				last_index - index);
		if (iocb->ki_flags & IOCB_NOWAIT)
			memalloc_noio_restore(flags);
		filemap_get_read_batch(mapping, index, last_index - 1, fbatch);
	}
	if (!folio_batch_count(fbatch)) {
		if (iocb->ki_flags & (IOCB_NOWAIT | IOCB_WAITQ))
			return -EAGAIN;
		err = filemap_create_folio(filp, mapping, iocb->ki_pos, fbatch);
		if (err == AOP_TRUNCATED_PAGE)
			goto retry;
		return err;
	}
```

* **`/* "last_index" is the index of the page beyond the end of the read */`**: 注释，解释 `last_index` 的含义。
* **`last_index = DIV_ROUND_UP(iocb->ki_pos + count, PAGE_SIZE);`**: 计算读取结束页的索引 `last_index`。`DIV_ROUND_UP` 宏表示向上取整的除法，确保包含读取范围内的所有页。
* **`retry:`**:  标签，用于错误处理时的重试。
* **`if (fatal_signal_pending(current))`**: 检查是否有致命信号待处理。如果有，则返回 `-EINTR` 错误，表示操作被信号中断。
* **`filemap_get_read_batch(mapping, index, last_index - 1, fbatch);`**: **尝试从页缓存批量获取 folio。**
    * `mapping`: 地址空间。
    * `index`: 起始页索引。
    * `last_index - 1`: 结束页索引 (包含)。
    * `fbatch`: folio 批量结构体。
    * `filemap_get_read_batch` 函数会尝试从页缓存中查找从 `index` 到 `last_index - 1` 范围内的 folio，并将找到的 folio 添加到 `fbatch` 中。**如果 folio 已经在页缓存中，并且状态合适 (例如，没有被锁定)，则可以快速获取到。**
* **`if (!folio_batch_count(fbatch))`**: 检查 `fbatch` 中是否获取到任何 folio。如果没有获取到 (页缓存未命中或没有合适的 folio)，则执行以下代码块。
    * **`if (iocb->ki_flags & IOCB_NOIO)`**: 检查 `IOCB_NOIO` 标志。如果设置了 `IOCB_NOIO`，表示不允许进行 I/O 操作，直接返回 `-EAGAIN` 错误 (资源暂时不可用)。这通常用于非阻塞的读取操作，如果数据不在缓存中，则立即返回错误，而不是等待 I/O。
    * **`if (iocb->ki_flags & IOCB_NOWAIT)`**: 检查 `IOCB_NOWAIT` 标志。如果设置了 `IOCB_NOWAIT`，表示非阻塞等待。
        * `flags = memalloc_noio_save();`: 保存当前的内存分配上下文，进入 `NOIO` 模式，限制某些可能导致阻塞的内存分配行为。
    * **`page_cache_sync_readahead(mapping, ra, filp, index, last_index - index);`**: **触发同步预读操作！**
        * `mapping`: 地址空间。
        * `ra`: 预读状态。
        * `filp`: 文件结构体。
        * `index`: 起始页索引。
        * `last_index - index`:  预读的页数。
        * `page_cache_sync_readahead` 函数会根据预读策略，**发起预读操作，将文件数据从磁盘读取到页缓存中。**  **这里就封装了 readahead 操作。**  "sync" 表示这个预读操作是同步的，但实际上预读本身通常是异步进行的，这里 "sync" 可能指的是触发预读的动作是同步的。
    * **`if (iocb->ki_flags & IOCB_NOWAIT)`**: 如果设置了 `IOCB_NOWAIT`，则恢复之前的内存分配上下文。
        * `memalloc_noio_restore(flags);`
    * **`filemap_get_read_batch(mapping, index, last_index - 1, fbatch);`**: **再次尝试从页缓存批量获取 folio。**  在预读之后，再次尝试获取，希望预读操作已经将需要的 folio 加载到页缓存中。
* **`if (!folio_batch_count(fbatch))`**: 再次检查 `fbatch` 中是否获取到任何 folio。如果仍然没有获取到 (即使预读之后仍然没有)，则执行以下代码块。
    * **`if (iocb->ki_flags & (IOCB_NOWAIT | IOCB_WAITQ))`**: 检查是否设置了 `IOCB_NOWAIT` 或 `IOCB_WAITQ` 标志。如果设置了，表示不允许阻塞等待，直接返回 `-EAGAIN` 错误。`IOCB_WAITQ` 可能表示操作被排队，不应该无限期等待。
    * **`err = filemap_create_folio(filp, mapping, iocb->ki_pos, fbatch);`**: **创建新的 folio 并尝试读取数据。**
        * `filp`: 文件结构体。
        * `mapping`: 地址空间。
        * `iocb->ki_pos`: 读取起始位置。
        * `fbatch`: folio 批量结构体。
        * `filemap_create_folio` 函数会创建一个新的 folio，并**尝试从磁盘读取数据填充这个 folio。**  **这里可能封装了 `read_folio` 操作，或者更底层的页面创建和 I/O 发起逻辑。**  具体的 I/O 操作可能会委托给地址空间 `mapping->a_ops` 中的操作。
    * **`if (err == AOP_TRUNCATED_PAGE)`**: 检查 `filemap_create_folio` 是否返回 `AOP_TRUNCATED_PAGE` 错误。这个错误可能表示在创建 folio 的过程中，文件被截断了。
        * `goto retry;`: 如果是 `AOP_TRUNCATED_PAGE` 错误，则跳转到 `retry` 标签，重新尝试获取 folio。这是一种处理文件截断的重试机制。
    * **`return err;`**: 返回 `filemap_create_folio` 的错误码。

```c
	folio = fbatch->folios[folio_batch_count(fbatch) - 1];
	if (folio_test_readahead(folio)) {
		err = filemap_readahead(iocb, filp, mapping, folio, last_index);
		if (err)
			goto err;
	}
	if (!folio_test_uptodate(folio)) {
		if ((iocb->ki_flags & IOCB_WAITQ) &&
		    folio_batch_count(fbatch) > 1)
			iocb->ki_flags |= IOCB_NOWAIT;
		err = filemap_update_page(iocb, mapping, count, folio,
					  need_uptodate);
		if (err)
			goto err;
	}
```

* **`folio = fbatch->folios[folio_batch_count(fbatch) - 1];`**: 获取 `fbatch` 中最后一个 folio。这里假设 `fbatch` 中至少有一个 folio (因为前面的 `if (!folio_batch_count(fbatch))` 已经处理了没有获取到 folio 的情况)。
* **`if (folio_test_readahead(folio))`**: 检查 folio 是否被标记为需要预读 (`readahead`)。`folio_test_readahead` 可能是检查 folio 的某个标志位。
    * **`err = filemap_readahead(iocb, filp, mapping, folio, last_index);`**: **触发异步预读操作！**
        * `iocb`, `filp`, `mapping`, `folio`, `last_index`: 参数传递。
        * `filemap_readahead` 函数会发起异步预读操作，预取后续可能需要的 folio。**这是另一种形式的 readahead，可能是针对当前 folio 的后续数据进行预读。**
        * `if (err) goto err;`: 如果 `filemap_readahead` 返回错误，则跳转到 `err` 标签处理错误。
* **`if (!folio_test_uptodate(folio))`**: 检查 folio 是否是 `Uptodate` 状态。`folio_test_uptodate` 可能是检查 folio 的某个标志位，判断数据是否已加载完成。
    * **`if ((iocb->ki_flags & IOCB_WAITQ) && folio_batch_count(fbatch) > 1)`**: 如果设置了 `IOCB_WAITQ` 并且 `fbatch` 中有多个 folio，则设置 `IOCB_NOWAIT` 标志。这可能是一种优化策略，在异步读取多个 folio 的情况下，如果已经读取了一些 folio，则后续的 folio 可以设置为非阻塞读取。
    * **`err = filemap_update_page(iocb, mapping, count, folio, need_uptodate);`**: **确保 folio 数据是最新的 (如果需要)。**
        * `iocb`, `mapping`, `count`, `folio`, `need_uptodate`: 参数传递。
        * `filemap_update_page` 函数负责确保 folio 的数据是最新的，并将其标记为 `Uptodate` 状态。**如果 folio 的数据还没有从磁盘读取，或者读取过程中发生错误，`filemap_update_page` 可能会发起实际的 I/O 读取操作，并等待 I/O 完成。**  **这可能是 `read_folio` 操作的另一种封装形式，或者是在 `filemap_create_folio` 之后，确保数据最终被加载到 folio 的步骤。**
        * `if (err) goto err;`: 如果 `filemap_update_page` 返回错误，则跳转到 `err` 标签处理错误。

```c
	trace_mm_filemap_get_pages(mapping, index, last_index - 1);
	return 0;
err:
	if (err < 0)
		folio_put(folio);
	if (likely(--fbatch->nr))
		return 0;
	if (err == AOP_TRUNCATED_PAGE)
		goto retry;
	return err;
}
```

* **`trace_mm_filemap_get_pages(mapping, index, last_index - 1);`**:  tracepoint，用于性能分析和调试，记录 `filemap_get_pages` 的调用信息。
* **`return 0;`**:  函数成功返回 0。
* **`err:`**:  错误处理标签。
    * **`if (err < 0) folio_put(folio);`**: 如果错误码 `err` 是负值 (表示错误)，则释放 `folio` 的引用计数。
    * **`if (likely(--fbatch->nr))`**:  减少 `fbatch` 中的 folio 计数器 `nr`。`likely()` 宏表示条件为真的可能性更高。如果减少计数器后仍然大于 0，则表示 `fbatch` 中还有其他 folio，直接返回 0 (可能表示部分成功，或者错误不影响整体操作)。
    * **`if (err == AOP_TRUNCATED_PAGE) goto retry;`**: 如果错误码是 `AOP_TRUNCATED_PAGE`，则跳转到 `retry` 标签，重新尝试获取 folio。
    * **`return err;`**:  返回错误码 `err`。

**总结 `filemap_get_pages`:**

`filemap_get_pages` 函数是 `filemap_read` 获取 folio 的核心函数。它的主要逻辑是：

1. **首先尝试从页缓存中批量获取 folio (`filemap_get_read_batch`).**
2. **如果页缓存未命中，则触发同步预读 (`page_cache_sync_readahead`).**
3. **再次尝试从页缓存获取 folio.**
4. **如果仍然未获取到，则创建新的 folio 并尝试读取数据 (`filemap_create_folio`).**
5. **对获取到的 folio 进行进一步处理，包括异步预读 (`filemap_readahead`) 和确保数据是最新的 (`filemap_update_page`).**
6. **处理各种错误情况，包括重试机制和错误返回。**

**`readahead` 和 `read_folio` 的封装位置:**

* **`readahead` 操作主要封装在 `page_cache_sync_readahead` 和 `filemap_readahead` 函数中。**  `page_cache_sync_readahead` 触发同步预读，而 `filemap_readahead` 触发异步预读。
* **`read_folio` 操作 (或者更准确地说，页面创建和数据读取) 主要封装在 `filemap_create_folio` 和 `filemap_update_page` 函数中。**  `filemap_create_folio` 负责创建新的 folio 并启动读取，`filemap_update_page` 负责确保 folio 数据是最新的，并在必要时发起读取或等待读取完成。

**需要进一步了解的地方:**

为了更深入地理解，我们可能需要查看以下函数的代码：

* `filemap_get_read_batch`
* `page_cache_sync_readahead`
* `filemap_create_folio`
* `filemap_readahead`
* `filemap_update_page`

特别是 `filemap_create_folio` 和 `filemap_update_page`，它们很可能是直接调用地址空间 `address_space` 的 `a_ops` 中的 `readpage` 或 `readpages` 操作，从而最终触发底层的 I/O 读取。

希望这个详细的解析对您有所帮助！如果您想继续深入研究这些函数，或者有其他问题，请随时告诉我。

