 好的，让我们来逐步解析Linux内核的 `filemap_read` 函数，并重点关注它是如何处理页面不在page cache中的情况，以及如何利用 readahead 和 address_space operations 来获取数据。

**函数概览**

`filemap_read` 函数的主要职责是从页缓存中读取数据，以满足用户空间的读取请求。它处理了从文件系统中读取数据到内存页缓存，然后将数据复制到用户空间缓冲区这一过程。这个函数是现代Linux内核中基于folio的I/O操作的核心接口之一。

**代码逐行解析**

```c
ssize_t filemap_read(struct kiocb *iocb, struct iov_iter *iter,
		ssize_t already_read)
{
	struct file *filp = iocb->ki_filp;
	struct file_ra_state *ra = &filp->f_ra;
	struct address_space *mapping = filp->f_mapping;
	struct inode *inode = mapping->host;
	struct folio_batch fbatch;
	int i, error = 0;
	bool writably_mapped;
	loff_t isize, end_offset;
	loff_t last_pos = ra->prev_pos;
```

* **`ssize_t filemap_read(struct kiocb *iocb, struct iov_iter *iter, ssize_t already_read)`**: 函数定义。
    * `ssize_t`: 返回值类型，表示读取的字节数，出错时返回负的错误码。
    * `struct kiocb *iocb`:  I/O 控制块 (I/O Control Block)，包含了本次 I/O 操作的上下文信息，例如文件描述符、偏移量等。
    * `struct iov_iter *iter`:  描述用户空间缓冲区的数据结构，用于分散/聚集 I/O，可以处理多个不连续的缓冲区。
    * `ssize_t already_read`:  已经读取的字节数。在某些情况下，调用者可能已经读取了一部分数据，这个参数用于累加总共读取的字节数。
* **`struct file *filp = iocb->ki_filp;`**: 从 `iocb` 中获取 `file` 结构体指针。`file` 结构体代表了打开的文件实例。
* **`struct file_ra_state *ra = &filp->f_ra;`**: 获取文件的预读状态结构体 `file_ra_state`。预读 (readahead) 是一种优化技术，预测程序可能接下来会读取哪些数据，提前加载到页缓存中。
* **`struct address_space *mapping = filp->f_mapping;`**: 获取文件的地址空间 `address_space`。地址空间关联了文件在页缓存中的映射关系，以及与文件相关的各种操作，例如读取、写入、预读等。
* **`struct inode *inode = mapping->host;`**: 从地址空间中获取 `inode` 结构体指针。`inode` 结构体代表了文件系统中的文件节点，包含了文件的元数据信息，例如大小、权限等。
* **`struct folio_batch fbatch;`**:  声明一个 `folio_batch` 结构体。`folio_batch` 用于批量管理 folio，folio 是现代内核中页缓存的基本单位，可以看作是页 (page) 的升级版，可以管理更大尺寸的内存区域。
* **`int i, error = 0;`**: 声明循环计数器 `i` 和错误码变量 `error`，初始化 `error` 为 0。
* **`bool writably_mapped;`**: 声明一个布尔变量 `writably_mapped`，用于标记地址空间是否被可写地映射到用户空间。
* **`loff_t isize, end_offset;`**: 声明 `loff_t` 类型的变量 `isize` 和 `end_offset`。`isize` 用于存储文件的大小，`end_offset` 用于计算本次读取操作的结束偏移量。
* **`loff_t last_pos = ra->prev_pos;`**: 从预读状态 `ra` 中获取上次读取的位置 `prev_pos`，并赋值给 `last_pos`。这用于判断是否在同一个 folio 内连续读取。

```c
	if (unlikely(iocb->ki_pos < 0))
		return -EINVAL;
	if (unlikely(iocb->ki_pos >= inode->i_sb->s_maxbytes))
		return 0;
	if (unlikely(!iov_iter_count(iter)))
		return 0;

	iov_iter_truncate(iter, inode->i_sb->s_maxbytes - iocb->ki_pos);
	folio_batch_init(&fbatch);
```

* **`if (unlikely(iocb->ki_pos < 0))`**: 检查读取偏移量 `iocb->ki_pos` 是否小于 0。如果是，则返回 `-EINVAL` 错误，表示参数无效。`unlikely()` 是一个宏，用于标记条件为假的可能性更高，帮助编译器优化分支预测。
* **`if (unlikely(iocb->ki_pos >= inode->i_sb->s_maxbytes))`**: 检查读取偏移量是否超过文件系统支持的最大文件大小 `s_maxbytes`。如果是，则返回 0，表示已经到达文件末尾。
* **`if (unlikely(!iov_iter_count(iter)))`**: 检查用户空间缓冲区是否为空，即 `iov_iter` 中是否还有需要读取的数据。如果为空，则返回 0，表示没有数据需要读取。
* **`iov_iter_truncate(iter, inode->i_sb->s_maxbytes - iocb->ki_pos);`**:  根据文件系统最大文件大小限制，截断 `iov_iter` 的读取长度。确保读取操作不会超出文件系统限制。
* **`folio_batch_init(&fbatch);`**: 初始化 `folio_batch` 结构体，准备用于批量管理 folio。

```c
	do {
		cond_resched();

		/*
		 * If we've already successfully copied some data, then we
		 * can no longer safely return -EIOCBQUEUED. Hence mark
		 * an async read NOWAIT at that point.
		 */
		if ((iocb->ki_flags & IOCB_WAITQ) && already_read)
			iocb->ki_flags |= IOCB_NOWAIT;

		if (unlikely(iocb->ki_pos >= i_size_read(inode)))
			break;

		error = filemap_get_pages(iocb, iter->count, &fbatch, false);
		if (error < 0)
			break;
```

* **`do { ... } while (iov_iter_count(iter) && iocb->ki_pos < isize && !error);`**:  主循环，持续读取数据直到以下条件之一满足：
    * 用户空间缓冲区 `iter` 中没有更多空间 (`iov_iter_count(iter)` 为 0)。
    * 读取位置 `iocb->ki_pos` 达到文件大小 `isize`。
    * 发生错误 (`error` 不为 0)。
* **`cond_resched();`**:  条件性地让出 CPU。如果当前进程需要等待资源或者已经运行了较长时间，则允许其他进程运行，提高系统公平性。
* **注释块**: 解释了异步读取和 `-EIOCBQUEUED` 的处理。如果已经成功读取了一些数据，则不能再返回 `-EIOCBQUEUED` (表示操作被排队)，因此将异步读取标记为 `NOWAIT`。
* **`if (unlikely(iocb->ki_pos >= i_size_read(inode)))`**: 再次检查读取位置是否超过文件当前大小。`i_size_read(inode)` 读取的是文件的逻辑大小，可能会在读取过程中被截断。如果超过文件大小，则跳出循环。
* **`error = filemap_get_pages(iocb, iter->count, &fbatch, false);`**: **关键函数！**  调用 `filemap_get_pages` 函数来获取需要的 folio。
    * `iocb`: I/O 控制块。
    * `iter->count`:  本次读取请求的字节数。
    * `&fbatch`:  指向 `folio_batch` 结构体的指针，用于存储获取到的 folio。
    * `false`:  `async_read` 参数，这里设置为 `false` 表示同步读取。
    * **`filemap_get_pages` 的作用就是根据当前的读取位置和请求长度，从页缓存中查找或读取所需的 folio。如果 folio 不在页缓存中，它会触发预读和页面调入操作。**  **这就是注释中提到的 "readahead and read_folio address_space operations" 的封装之处。**
* **`if (error < 0)`**: 检查 `filemap_get_pages` 是否返回错误。如果返回错误，则跳出循环。

```c
		/*
		 * i_size must be checked after we know the pages are Uptodate.
		 *
		 * Checking i_size after the check allows us to calculate
		 * the correct value for "nr", which means the zero-filled
		 * part of the page is not copied back to userspace (unless
		 * another truncate extends the file - this is desired though).
		 */
		isize = i_size_read(inode);
		if (unlikely(iocb->ki_pos >= isize))
			goto put_folios;
		end_offset = min_t(loff_t, isize, iocb->ki_pos + iter->count);
```

* **注释块**: 解释了为什么要在获取 folio 之后再次检查 `i_size`。这是为了确保在页面被标记为 `Uptodate` (数据已加载完成) 之后，文件大小的检查是准确的。这样可以避免将页面的零填充部分错误地复制到用户空间。
* **`isize = i_size_read(inode);`**: 再次读取文件大小 `isize`，确保获取到最新的文件大小。
* **`if (unlikely(iocb->ki_pos >= isize))`**: 再次检查读取位置是否超过文件大小。如果超过，则跳转到 `put_folios` 标签，释放已获取的 folio 并结束读取。
* **`end_offset = min_t(loff_t, isize, iocb->ki_pos + iter->count);`**: 计算本次读取操作的结束偏移量 `end_offset`。取文件大小 `isize` 和请求读取的结束位置 `iocb->ki_pos + iter->count` 的最小值，防止读取超出文件末尾。

```c
		/*
		 * Once we start copying data, we don't want to be touching any
		 * cachelines that might be contended:
		 */
		writably_mapped = mapping_writably_mapped(mapping);

		/*
		 * When a read accesses the same folio several times, only
		 * mark it as accessed the first time.
		 */
		if (!pos_same_folio(iocb->ki_pos, last_pos - 1,
				    fbatch.folios[0]))
			folio_mark_accessed(fbatch.folios[0]);
```

* **注释块**:  解释了在开始复制数据后，要避免访问可能被竞争的缓存行。
* **`writably_mapped = mapping_writably_mapped(mapping);`**:  检查地址空间是否被可写地映射到用户空间。如果是，则需要进行额外的缓存一致性处理。
* **注释块**: 解释了对于同一个 folio 的多次读取，只需要在第一次访问时标记为已访问。
* **`if (!pos_same_folio(iocb->ki_pos, last_pos - 1, fbatch.folios[0]))`**:  判断当前读取位置 `iocb->ki_pos` 是否与上次读取位置 `last_pos - 1` 在同一个 folio 内。`pos_same_folio` 宏用于判断两个偏移量是否在同一个 folio 内。
* **`folio_mark_accessed(fbatch.folios[0]);`**: 如果当前读取位置不在同一个 folio 内 (或者第一次读取)，则标记第一个 folio 为已访问。`folio_mark_accessed` 函数会更新 folio 的访问时间戳，用于页缓存的 LRU (Least Recently Used) 回收算法。

```c
		for (i = 0; i < folio_batch_count(&fbatch); i++) {
			struct folio *folio = fbatch.folios[i];
			size_t fsize = folio_size(folio);
			size_t offset = iocb->ki_pos & (fsize - 1);
			size_t bytes = min_t(loff_t, end_offset - iocb->ki_pos,
					     fsize - offset);
			size_t copied;

			if (end_offset < folio_pos(folio))
				break;
			if (i > 0)
				folio_mark_accessed(folio);
			/*
			 * If users can be writing to this folio using arbitrary
			 * virtual addresses, take care of potential aliasing
			 * before reading the folio on the kernel side.
			 */
			if (writably_mapped)
				flush_dcache_folio(folio);

			copied = copy_folio_to_iter(folio, offset, bytes, iter);

			already_read += copied;
			iocb->ki_pos += copied;
			last_pos = iocb->ki_pos;

			if (copied < bytes) {
				error = -EFAULT;
				break;
			}
		}
```

* **`for (i = 0; i < folio_batch_count(&fbatch); i++) { ... }`**: 循环遍历 `folio_batch` 中的每个 folio。
* **`struct folio *folio = fbatch.folios[i];`**: 获取当前 folio。
* **`size_t fsize = folio_size(folio);`**: 获取 folio 的大小。
* **`size_t offset = iocb->ki_pos & (fsize - 1);`**: 计算在 folio 内的偏移量 `offset`。使用位运算 `& (fsize - 1)` 等价于取模 `% fsize`，但效率更高，因为 folio 大小通常是 2 的幂。
* **`size_t bytes = min_t(loff_t, end_offset - iocb->ki_pos, fsize - offset);`**: 计算本次需要从当前 folio 复制的字节数 `bytes`。取剩余需要读取的字节数 `end_offset - iocb->ki_pos` 和当前 folio 剩余可读字节数 `fsize - offset` 的最小值。
* **`size_t copied;`**: 声明变量 `copied` 用于存储实际复制的字节数。
* **`if (end_offset < folio_pos(folio))`**: 检查结束偏移量 `end_offset` 是否小于当前 folio 的起始位置 `folio_pos(folio)`。如果是，则说明已经读取完所需数据，跳出循环。
* **`if (i > 0) folio_mark_accessed(folio);`**:  如果不是第一个 folio (i > 0)，则标记为已访问。因为第一个 folio 在循环外已经标记过。
* **注释块**: 解释了如果用户空间可以写入这个 folio (通过可写映射)，需要处理潜在的别名问题。
* **`if (writably_mapped) flush_dcache_folio(folio);`**: 如果地址空间被可写地映射，则刷新 folio 的数据缓存 (D-Cache)。`flush_dcache_folio` 函数确保内核读取到最新的数据，避免缓存一致性问题。
* **`copied = copy_folio_to_iter(folio, offset, bytes, iter);`**: **数据复制的核心函数！**  调用 `copy_folio_to_iter` 函数将 folio 中的数据复制到用户空间缓冲区 `iter`。
    * `folio`:  源 folio。
    * `offset`:  在 folio 内的起始偏移量。
    * `bytes`:  要复制的字节数。
    * `iter`:  目标用户空间缓冲区。
    * `copy_folio_to_iter` 函数负责将 folio 中的数据复制到 `iov_iter` 描述的用户空间缓冲区，并更新 `iov_iter` 的状态。
* **`already_read += copied;`**:  累加已读取的字节数。
* **`iocb->ki_pos += copied;`**:  更新 `iocb` 中的读取位置 `ki_pos`。
* **`last_pos = iocb->ki_pos;`**: 更新 `last_pos` 为最新的读取位置。
* **`if (copied < bytes)`**: 检查实际复制的字节数 `copied` 是否小于期望复制的字节数 `bytes`。如果小于，则表示复制过程中发生错误 (例如用户空间缓冲区无效)，设置错误码为 `-EFAULT` 并跳出循环。

```c
put_folios:
		for (i = 0; i < folio_batch_count(&fbatch); i++)
			folio_put(fbatch.folios[i]);
		folio_batch_init(&fbatch);
	} while (iov_iter_count(iter) && iocb->ki_pos < isize && !error);
```

* **`put_folios:`**:  标签，用于跳转到这里释放 folio。
* **`for (i = 0; i < folio_batch_count(&fbatch); i++) folio_put(fbatch.folios[i]);`**: 循环遍历 `folio_batch` 中的每个 folio，并调用 `folio_put` 函数释放 folio 的引用计数。`folio_put` 类似于页缓存的 `page_cache_release`，用于减少 folio 的引用计数，当引用计数降为 0 时，folio 可能会被释放回页缓存。
* **`folio_batch_init(&fbatch);`**: 重新初始化 `folio_batch` 结构体，为下一次循环迭代做准备。
* **` } while (iov_iter_count(iter) && iocb->ki_pos < isize && !error);`**:  循环条件，继续循环直到用户空间缓冲区已满、读取位置达到文件末尾或发生错误。

```c
	file_accessed(filp);
	ra->prev_pos = last_pos;
	return already_read ? already_read : error;
}
EXPORT_SYMBOL_GPL(filemap_read);
```

* **`file_accessed(filp);`**:  更新文件的访问时间戳。
* **`ra->prev_pos = last_pos;`**:  更新预读状态 `ra` 中的 `prev_pos` 为最新的读取位置 `last_pos`，用于下一次预读的判断。
* **`return already_read ? already_read : error;`**:  返回值。如果 `already_read` 大于 0，则返回已读取的字节数 `already_read`。否则，返回错误码 `error`。使用三元运算符简洁地处理了成功读取和出错的情况。
* **`EXPORT_SYMBOL_GPL(filemap_read);`**:  将 `filemap_read` 函数导出为内核符号，并声明使用 GPL 协议。这使得其他内核模块可以调用 `filemap_read` 函数。

**readahead 和 read_folio address_space operations 的封装**

正如您所指出的，注释中提到如果页面不在 page cache 中，`filemap_read` 会使用 readahead 和 read_folio address_space operations 来获取数据。**这些操作的核心封装在 `filemap_get_pages` 函数中。**

让我们更深入地看一下 `filemap_get_pages` (虽然我们没有 `filemap_get_pages` 的源代码，但我们可以根据其功能和常见的内核实现来推断)：

`filemap_get_pages` 的大致工作流程可能如下：

1. **计算需要的 folio 范围:**  根据 `iocb->ki_pos` 和 `iter->count` 计算本次读取操作需要哪些 folio。
2. **检查页缓存:**  对于每个需要的 folio，检查它是否已经在页缓存中。
3. **处理页缓存命中:** 如果 folio 已经在页缓存中，并且是 `Uptodate` 状态 (数据已加载完成)，则直接使用该 folio。
4. **处理页缓存未命中 (需要读取):** 如果 folio 不在页缓存中，或者不是 `Uptodate` 状态，则需要从磁盘读取数据。
    * **触发预读 (readahead):**  `filemap_get_pages` 可能会根据预读策略 (由 `file_ra_state` 管理) 决定是否需要进行预读。预读会提前读取后续可能需要访问的 folio，以提高性能。预读操作通常会调用地址空间 `address_space` 的 `readahead` 操作 (例如 `mapping->a_ops->readahead`)。
    * **获取或创建 folio:**  `filemap_get_pages` 会尝试获取或创建一个新的 folio 来存储即将读取的数据。
    * **发起 I/O 读取:**  调用地址空间 `address_space` 的 `read_folio` 操作 (例如 `mapping->a_ops->read_folio`) 来发起实际的 I/O 读取操作，将数据从磁盘读取到 folio 中。`read_folio` 操作通常会与底层的块设备驱动交互，完成数据的物理读取。
    * **等待 I/O 完成:**  `filemap_get_pages` 需要等待 I/O 读取操作完成，确保 folio 中的数据是 `Uptodate` 的。
5. **批量返回 folio:**  将获取到的 (可能从页缓存中获取，也可能是新读取的) folio 存储在 `fbatch` 中，并返回给 `filemap_read` 函数。

**总结**

`filemap_read` 函数是内核中读取文件数据的核心接口，它负责从页缓存中读取数据，并在数据不在页缓存时，通过 `filemap_get_pages` 函数触发预读和页面调入操作。`filemap_get_pages` 函数内部会利用地址空间 `address_space` 提供的 `readahead` 和 `read_folio` 操作来完成页面的预取和读取，从而实现了高效的文件数据读取。

希望这个详细的解析能够帮助您理解 `filemap_read` 函数的工作原理以及它如何处理页缓存未命中的情况。如果您有任何其他问题，请随时提出。

