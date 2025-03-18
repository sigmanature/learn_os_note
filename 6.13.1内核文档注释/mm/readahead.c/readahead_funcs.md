 好的，我们来逐步解析您提供的这些与预读 (readahead) 相关的内核函数：`page_cache_sync_readahead`, `page_cache_sync_ra`, `force_page_cache_ra`, `do_page_cache_ra`, 和 `page_cache_ra_unbounded`。 这些函数共同实现了同步预读的逻辑，让我们深入了解它们是如何协同工作的。

**1. `page_cache_sync_readahead` 函数解析**

```c
static inline
void page_cache_sync_readahead(struct address_space *mapping,
		struct file_ra_state *ra, struct file *file, pgoff_t index,
		unsigned long req_count)
{
	DEFINE_READAHEAD(ractl, file, ra, mapping, index);
	page_cache_sync_ra(&ractl, req_count);
}
```

* **`static inline void page_cache_sync_readahead(...)`**: 函数定义。
    * `static inline`: 静态内联函数。
    * `void`:  无返回值。
    * 参数：
        * `struct address_space *mapping`: 文件的地址空间。
        * `struct file_ra_state *ra`: 文件的预读状态结构体。
        * `struct file *file`: 文件结构体。
        * `pgoff_t index`:  预读起始页的索引。
        * `unsigned long req_count`:  请求读取的页数 (或字节数，取决于上下文，这里是页数)。
* **`DEFINE_READAHEAD(ractl, file, ra, mapping, index);`**:  宏调用，用于初始化 `readahead_control` 结构体 `ractl`。
    * `DEFINE_READAHEAD` 宏 (未提供代码，但可以推断其功能)  很可能负责分配和初始化一个 `readahead_control` 结构体，并将传入的参数 `file`, `ra`, `mapping`, `index` 赋值给 `ractl` 结构体的相应成员。`readahead_control` 结构体用于管理预读操作的上下文信息。
* **`page_cache_sync_ra(&ractl, req_count);`**:  调用 `page_cache_sync_ra` 函数，执行实际的同步预读操作。将初始化好的 `readahead_control` 结构体 `ractl` 的地址和请求的页数 `req_count` 传递给 `page_cache_sync_ra`。

**总结 `page_cache_sync_readahead`:**

`page_cache_sync_readahead` 是同步预读的入口函数，它负责初始化预读控制结构体 `readahead_control`，然后调用 `page_cache_sync_ra` 函数来执行核心的预读逻辑。它本身只是一个简单的包装函数，主要工作委托给了 `page_cache_sync_ra`。

**2. `page_cache_sync_ra` 函数解析**

```c
void page_cache_sync_ra(struct readahead_control *ractl,
		unsigned long req_count)
{
	pgoff_t index = readahead_index(ractl);/*直接返回ractl->index表示这是预读请求的第一个page index*/
	bool do_forced_ra = ractl->file && (ractl->file->f_mode & FMODE_RANDOM);
	struct file_ra_state *ra = ractl->ra;
	unsigned long max_pages, contig_count;
	pgoff_t prev_index, miss;
```

* **`void page_cache_sync_ra(struct readahead_control *ractl, unsigned long req_count)`**: 函数定义。
    * `void`: 无返回值。
    * 参数：
        * `struct readahead_control *ractl`: 预读控制结构体指针。
        * `unsigned long req_count`: 请求读取的页数。
* **`pgoff_t index = readahead_index(ractl);`**: 获取预读起始页索引 `index`。`readahead_index(ractl)` 宏 (未提供代码，但注释说明)  很可能直接返回 `ractl->index`，表示这是预读请求的第一个页索引。
* **`bool do_forced_ra = ractl->file && (ractl->file->f_mode & FMODE_RANDOM);`**:  判断是否进行强制预读。
    * `ractl->file && (ractl->file->f_mode & FMODE_RANDOM)`:  如果 `ractl->file` 存在 (表示与文件关联) 并且文件以随机访问模式 (`FMODE_RANDOM`) 打开，则 `do_forced_ra` 为 `true`。对于随机访问模式，传统的顺序预读策略可能效果不佳，因此可能需要采用不同的预读策略 (强制预读)。
* **`struct file_ra_state *ra = ractl->ra;`**: 获取文件的预读状态结构体 `ra`。
* **`unsigned long max_pages, contig_count;`**: 声明变量 `max_pages` 和 `contig_count`。`max_pages` 用于存储最大预读页数，`contig_count` 用于存储连续缓存页计数。
* **`pgoff_t prev_index, miss;`**: 声明变量 `prev_index` 和 `miss`。`prev_index` 用于存储上次读取的页索引，`miss` 用于存储上次缓存未命中的页索引。

```c
	/*
	 * Even if readahead is disabled, issue this request as readahead
	 * as we'll need it to satisfy the requested range. The forced
	 * readahead will do the right thing and limit the read to just the
	 * requested range, which we'll set to 1 page for this case.
	 */
	if (!ra->ra_pages || blk_cgroup_congested()) {
		if (!ractl->file)
			return;
		req_count = 1;
		do_forced_ra = true;
	}
```

* **注释块**: 解释了即使预读被禁用，也要作为预读请求发出，因为需要满足请求的范围。强制预读会限制读取范围为请求的范围，这里设置为 1 页。
* **`if (!ra->ra_pages || blk_cgroup_congested())`**:  检查预读是否被禁用或块设备组是否拥塞。
    * `!ra->ra_pages`:  检查 `ra->ra_pages` 是否为 0。`ra->ra_pages` 存储了预读窗口大小 (页数)。如果为 0，表示预读被禁用。
    * `blk_cgroup_congested()`:  检查块设备组是否拥塞。如果拥塞，可能需要减少预读量或禁用预读。
    * 如果以上条件之一为真，则执行以下代码块。
        * `if (!ractl->file) return;`: 如果 `ractl->file` 为空 (不与文件关联)，则直接返回。
        * `req_count = 1;`: 将请求页数 `req_count` 设置为 1。即使预读被禁用，仍然至少读取 1 页来满足当前请求。
        * `do_forced_ra = true;`: 设置 `do_forced_ra` 为 `true`，强制进行预读。
* **`/* be dumb */`**: 注释，表示接下来的逻辑比较简单直接。
* **`if (do_forced_ra)`**: 如果需要强制预读。
    * `force_page_cache_ra(ractl, req_count);`: 调用 `force_page_cache_ra` 函数执行强制预读，并返回。

```c
	max_pages = ractl_max_pages(ractl, req_count);
	prev_index = (unsigned long long)ra->prev_pos >> PAGE_SHIFT;
	/*
	 * A start of file, oversized read, or sequential cache miss:
	 * trivial case: (index - prev_index) == 1
	 * unaligned reads: (index - prev_index) == 0
	 */
	if (!index || req_count > max_pages || index - prev_index <= 1UL) {
		ra->start = index;
		ra->size = get_init_ra_size(req_count, max_pages);
		ra->async_size = ra->size > req_count ? ra->size - req_count :
							ra->size >> 1;
		goto readit;
	}
```

* **`max_pages = ractl_max_pages(ractl, req_count);`**: 计算最大预读页数 `max_pages`。`ractl_max_pages(ractl, req_count)` 宏 (未提供代码)  很可能根据 `ractl` 和 `req_count` 计算允许的最大预读页数，可能考虑到预读窗口大小、内存限制等因素。
* **`prev_index = (unsigned long long)ra->prev_pos >> PAGE_SHIFT;`**: 计算上次读取的页索引 `prev_index`。从预读状态 `ra` 中获取上次读取的位置 `prev_pos`，并转换为页索引。
* **注释块**: 解释了几种触发简单预读策略的情况：文件开始、超大读取、或顺序缓存未命中 (trivial case: `index - prev_index == 1`)，以及非对齐读取 (`index - prev_index == 0`)。
* **`if (!index || req_count > max_pages || index - prev_index <= 1UL)`**:  判断是否满足简单预读策略的条件。
    * `!index`:  如果 `index` 为 0，表示从文件开头开始读取。
    * `req_count > max_pages`: 如果请求页数 `req_count` 大于最大预读页数 `max_pages`，表示超大读取。
    * `index - prev_index <= 1UL`: 如果当前页索引 `index` 与上次读取的页索引 `prev_index` 的差值小于等于 1，表示顺序读取或非对齐读取。
    * 如果满足以上条件之一，则执行以下代码块。
        * `ra->start = index;`: 设置预读起始页索引 `ra->start` 为当前页索引 `index`。
        * `ra->size = get_init_ra_size(req_count, max_pages);`: 计算预读大小 `ra->size`。`get_init_ra_size(req_count, max_pages)` 函数 (未提供代码)  很可能根据请求页数 `req_count` 和最大预读页数 `max_pages` 计算初始预读大小。
        * `ra->async_size = ra->size > req_count ? ra->size - req_count : ra->size >> 1;`: 计算异步预读大小 `ra->async_size`。如果预读大小大于请求大小，则异步预读大小为预读大小减去请求大小，否则为预读大小的一半。
        * `goto readit;`: 跳转到 `readit` 标签，执行实际的预读操作。

```c
	/*
	 * Query the page cache and look for the traces(cached history pages)
	 * that a sequential stream would leave behind.
	 */
	rcu_read_lock();
	miss = page_cache_prev_miss(ractl->mapping, index - 1, max_pages);
	rcu_read_unlock();
	contig_count = index - miss - 1;
	/*
	 * Standalone, small random read. Read as is, and do not pollute the
	 * readahead state.
	 */
	if (contig_count <= req_count) {
		do_page_cache_ra(ractl, req_count, 0);
		return;
	}
	/*
	 * File cached from the beginning:
	 * it is a strong indication of long-run stream (or whole-file-read)
	 */
	if (miss == ULONG_MAX)
		contig_count *= 2;
	ra->start = index;
	ra->size = min(contig_count + req_count, max_pages);
	ra->async_size = 1;
readit:
	ractl->_index = ra->start;
	page_cache_ra_order(ractl, ra, 0);
}
```

* **注释块**: 解释了查询页缓存，查找顺序流留下的痕迹 (缓存的历史页)。
* **`rcu_read_lock(); ... rcu_read_unlock();`**:  RCU (Read-Copy-Update) 读锁，用于保护页缓存的并发访问。
* **`miss = page_cache_prev_miss(ractl->mapping, index - 1, max_pages);`**:  查询页缓存中上次缓存未命中的页索引 `miss`。`page_cache_prev_miss` 函数 (未提供代码)  很可能在页缓存中查找当前页索引 `index - 1` 之前的最近一次缓存未命中的页索引，用于判断是否是顺序流。
* **`contig_count = index - miss - 1;`**: 计算连续缓存页计数 `contig_count`。表示从 `miss + 1` 到 `index - 1` 之间的页都是缓存命中的，即连续缓存页的数量。
* **注释块**: 解释了独立的小型随机读取，直接读取请求的页数，不污染预读状态。
* **`if (contig_count <= req_count)`**: 如果连续缓存页计数 `contig_count` 小于等于请求页数 `req_count`，可能被认为是随机读取或顺序性较差的读取。
    * `do_page_cache_ra(ractl, req_count, 0);`: 调用 `do_page_cache_ra` 函数执行预读，预读页数为请求页数 `req_count`，lookahead 大小为 0。
    * `return;`: 返回。
* **注释块**: 解释了如果 `miss == ULONG_MAX`，表示文件从头开始缓存，强烈暗示长运行流 (或全文件读取)。
* **`if (miss == ULONG_MAX) contig_count *= 2;`**: 如果 `miss` 为 `ULONG_MAX`，则将连续缓存页计数 `contig_count` 乘以 2，可能用于扩大预读范围。
* **`ra->start = index;`**: 设置预读起始页索引 `ra->start` 为当前页索引 `index`。
* **`ra->size = min(contig_count + req_count, max_pages);`**: 计算预读大小 `ra->size`。预读大小为连续缓存页计数 `contig_count` 加上请求页数 `req_count`，但不能超过最大预读页数 `max_pages`。
* **`ra->async_size = 1;`**: 设置异步预读大小 `ra->async_size` 为 1。
* **`readit:`**: 标签。
* **`ractl->_index = ra->start;`**: 设置 `ractl` 中的当前预读索引 `_index` 为预读起始页索引 `ra->start`。
* **`page_cache_ra_order(ractl, ra, 0);`**: **调用 `page_cache_ra_order` 函数执行实际的预读操作。**  `page_cache_ra_order` 函数 (未提供代码)  很可能是负责根据预读控制结构体 `ractl` 和预读状态 `ra`，以及预读顺序 (这里是 0，可能表示顺序预读)  来发起实际的 I/O 读取操作。

**总结 `page_cache_sync_ra`:**

`page_cache_sync_ra` 函数是同步预读的核心逻辑函数。它根据文件访问模式、预读状态、缓存命中情况等因素，决定预读策略和预读大小。它区分了强制预读和普通预读，并根据不同的情况调用 `force_page_cache_ra` 或 `page_cache_ra_order` 函数来执行实际的预读操作。它还利用页缓存的历史信息来优化预读策略，例如通过 `page_cache_prev_miss` 函数查找上次缓存未命中的位置，从而判断是否是顺序流。

**3. `force_page_cache_ra` 函数解析**

```c
/*
 * Chunk the readahead into 2 megabyte units, so that we don't pin too much
 * memory at once.
 */
void force_page_cache_ra(struct readahead_control *ractl,
		unsigned long nr_to_read)
{
	struct address_space *mapping = ractl->mapping;
	struct file_ra_state *ra = ractl->ra;
	struct backing_dev_info *bdi = inode_to_bdi(mapping->host);
	unsigned long max_pages;
```

* **注释块**: 解释了将预读分块为 2MB 单位，以避免一次性锁定过多内存。
* **`void force_page_cache_ra(struct readahead_control *ractl, unsigned long nr_to_read)`**: 函数定义。
    * `void`: 无返回值。
    * 参数：
        * `struct readahead_control *ractl`: 预读控制结构体指针。
        * `unsigned long nr_to_read`: 需要读取的页数。
* **`struct address_space *mapping = ractl->mapping;`**: 获取地址空间 `mapping`。
* **`struct file_ra_state *ra = ractl->ra;`**: 获取预读状态 `ra`。
* **`struct backing_dev_info *bdi = inode_to_bdi(mapping->host);`**: 获取后备设备信息 `bdi`。`inode_to_bdi` 函数 (未提供代码)  很可能根据 inode 获取关联的后备设备信息，例如块设备驱动信息。
* **`unsigned long max_pages;`**: 声明变量 `max_pages`。

```c
	if (unlikely(!mapping->a_ops->read_folio && !mapping->a_ops->readahead))
		return;

	/*
	 * If the request exceeds the readahead window, allow the read to
	 * be up to the optimal hardware IO size
	 */
	max_pages = max_t(unsigned long, bdi->io_pages, ra->ra_pages);
	nr_to_read = min_t(unsigned long, nr_to_read, max_pages);
	while (nr_to_read) {
		unsigned long this_chunk = (2 * 1024 * 1024) / PAGE_SIZE;

		if (this_chunk > nr_to_read)
			this_chunk = nr_to_read;
		do_page_cache_ra(ractl, this_chunk, 0);

		nr_to_read -= this_chunk;
	}
}
```

* **`if (unlikely(!mapping->a_ops->read_folio && !mapping->a_ops->readahead))`**: 检查地址空间是否支持 `read_folio` 和 `readahead` 操作。如果都不支持，则直接返回，不做预读。`unlikely()` 宏表示条件为假的可能性更高。
* **注释块**: 解释了如果请求超过预读窗口，允许读取到硬件最优 I/O 大小。
* **`max_pages = max_t(unsigned long, bdi->io_pages, ra->ra_pages);`**: 计算最大预读页数 `max_pages`。取后备设备信息 `bdi->io_pages` (硬件最优 I/O 大小，以页为单位) 和预读窗口大小 `ra->ra_pages` 的最大值。
* **`nr_to_read = min_t(unsigned long, nr_to_read, max_pages);`**:  限制需要读取的页数 `nr_to_read` 不超过 `max_pages`。
* **`while (nr_to_read)`**:  循环，直到 `nr_to_read` 为 0。
    * **`unsigned long this_chunk = (2 * 1024 * 1024) / PAGE_SIZE;`**: 计算当前分块大小 `this_chunk`。设置为 2MB / PAGE_SIZE，即 2MB 的页数。
    * **`if (this_chunk > nr_to_read) this_chunk = nr_to_read;`**: 如果当前分块大小 `this_chunk` 大于剩余需要读取的页数 `nr_to_read`，则将 `this_chunk` 设置为 `nr_to_read`，确保不超过剩余页数。
    * **`do_page_cache_ra(ractl, this_chunk, 0);`**: **调用 `do_page_cache_ra` 函数执行实际的预读操作，预读页数为 `this_chunk`，lookahead 大小为 0。**
    * **`nr_to_read -= this_chunk;`**:  减少剩余需要读取的页数 `nr_to_read`。

**总结 `force_page_cache_ra`:**

`force_page_cache_ra` 函数用于执行强制预读。它将预读请求分块为 2MB 的单位，并循环调用 `do_page_cache_ra` 函数来执行每个分块的预读。这样做可能是为了限制一次性预读的内存量，避免锁定过多内存。它还考虑了硬件最优 I/O 大小和预读窗口大小，并确保预读操作不会超过这些限制。

**4. `do_page_cache_ra` 函数解析**

```c
/*
 * do_page_cache_ra() actually reads a chunk of disk.  It allocates
 * the pages first, then submits them for I/O. This avoids the very bad
 * behaviour which would occur if page allocations are causing VM writeback.
 * We really don't want to intermingle reads and writes like that.
 */
static void do_page_cache_ra(struct readahead_control *ractl,
		unsigned long nr_to_read, unsigned long lookahead_size)
{
	struct inode *inode = ractl->mapping->host;
	unsigned long index = readahead_index(ractl);
	loff_t isize = i_size_read(inode);
	pgoff_t end_index;	/* The last page we want to read */
```

* **注释块**: 解释了 `do_page_cache_ra` 函数实际读取磁盘块。它先分配页，然后提交 I/O。这样做是为了避免页分配导致 VM 回写，从而避免读写操作交错，影响性能。
* **`static void do_page_cache_ra(struct readahead_control *ractl, unsigned long nr_to_read, unsigned long lookahead_size)`**: 函数定义。
    * `static void`: 静态函数，无返回值。
    * 参数：
        * `struct readahead_control *ractl`: 预读控制结构体指针。
        * `unsigned long nr_to_read`: 需要读取的页数。
        * `unsigned long lookahead_size`: lookahead 大小 (用于后续预读，这里暂时为 0)。
* **`struct inode *inode = ractl->mapping->host;`**: 获取 inode 结构体指针。
* **`unsigned long index = readahead_index(ractl);`**: 获取预读起始页索引 `index`。
* **`loff_t isize = i_size_read(inode);`**: 读取文件大小 `isize`。`i_size_read` 函数 (未提供代码)  很可能读取 inode 中的文件大小信息。
* **`pgoff_t end_index;`**: 声明变量 `end_index`，用于存储文件结束页索引。
* **`/* The last page we want to read */`**: 注释，解释 `end_index` 的含义。

```c
	if (isize == 0)
		return;

	end_index = (isize - 1) >> PAGE_SHIFT;
	if (index > end_index)
		return;
	/* Don't read past the page containing the last byte of the file */
	if (nr_to_read > end_index - index)
		nr_to_read = end_index - index + 1;

	page_cache_ra_unbounded(ractl, nr_to_read, lookahead_size);
}
```

* **`if (isize == 0) return;`**: 如果文件大小 `isize` 为 0，则直接返回，不做预读。
* **`end_index = (isize - 1) >> PAGE_SHIFT;`**: 计算文件结束页索引 `end_index`。将文件大小减 1 (因为索引从 0 开始)，然后右移 `PAGE_SHIFT` 位，得到文件最后一页的索引。
* **`if (index > end_index) return;`**: 如果预读起始页索引 `index` 大于文件结束页索引 `end_index`，则直接返回，不做预读 (可能已经超出文件范围)。
* **注释块**: 解释了不要读取超过包含文件最后一个字节的页。
* **`if (nr_to_read > end_index - index)`**: 如果需要读取的页数 `nr_to_read` 大于从起始页索引 `index` 到结束页索引 `end_index` 的页数，则需要调整 `nr_to_read`。
    * `nr_to_read = end_index - index + 1;`: 将 `nr_to_read` 限制为从起始页索引 `index` 到结束页索引 `end_index` 的页数，确保预读不会超出文件末尾。
* **`page_cache_ra_unbounded(ractl, nr_to_read, lookahead_size);`**: **调用 `page_cache_ra_unbounded` 函数执行实际的无界预读操作。**  将预读控制结构体 `ractl`、需要读取的页数 `nr_to_read` 和 lookahead 大小 `lookahead_size` 传递给 `page_cache_ra_unbounded`。

**总结 `do_page_cache_ra`:**

`do_page_cache_ra` 函数负责限制预读范围，确保预读操作不会超出文件末尾。它首先检查文件大小，计算文件结束页索引，然后根据文件大小和请求的预读范围，调整实际需要预读的页数。最后，它调用 `page_cache_ra_unbounded` 函数来执行实际的无界预读操作。

**5. `page_cache_ra_unbounded` 函数解析**

```c
/**
 * page_cache_ra_unbounded - Start unchecked readahead.
 * @ractl: Readahead control.
 * @nr_to_read: The number of pages to read.
 * @lookahead_size: Where to start the next readahead.
 *
 * This function is for filesystems to call when they want to start
 * readahead beyond a file's stated i_size.  This is almost certainly
 * not the function you want to call.  Use page_cache_async_readahead()
 * or page_cache_sync_readahead() instead.
 *
 * Context: File is referenced by caller.  Mutexes may be held by caller.
 * May sleep, but will not reenter filesystem to reclaim memory.
 */
void page_cache_ra_unbounded(struct readahead_control *ractl,
		unsigned long nr_to_read, unsigned long lookahead_size)
{
	struct address_space *mapping = ractl->mapping;
	unsigned long index = readahead_index(ractl);
	gfp_t gfp_mask = readahead_gfp_mask(mapping);
	unsigned long mark = ULONG_MAX, i = 0;
	unsigned int min_nrpages = mapping_min_folio_nrpages(mapping);
```

* **注释块**: 详细描述了 `page_cache_ra_unbounded` 函数的作用：启动无界预读。强调这个函数通常不应该直接调用，而是应该使用 `page_cache_async_readahead` 或 `page_cache_sync_readahead`。并说明了上下文和可能睡眠的特性。
* **`void page_cache_ra_unbounded(struct readahead_control *ractl, unsigned long nr_to_read, unsigned long lookahead_size)`**: 函数定义。
    * `void`: 无返回值。
    * 参数：
        * `struct readahead_control *ractl`: 预读控制结构体指针。
        * `unsigned long nr_to_read`: 需要读取的页数。
        * `unsigned long lookahead_size`: lookahead 大小 (用于标记预读页)。
* **`struct address_space *mapping = ractl->mapping;`**: 获取地址空间 `mapping`。
* **`unsigned long index = readahead_index(ractl);`**: 获取预读起始页索引 `index`。
* **`gfp_t gfp_mask = readahead_gfp_mask(mapping);`**: 获取内存分配标志 `gfp_mask`。`readahead_gfp_mask(mapping)` 宏 (未提供代码)  很可能根据地址空间 `mapping` 获取适合预读操作的内存分配标志，例如 `__GFP_NOFS` 等。
* **`unsigned long mark = ULONG_MAX, i = 0;`**: 初始化变量 `mark` 和 `i`。`mark` 用于标记预读页的索引，`i` 是循环计数器。
* **`unsigned int min_nrpages = mapping_min_folio_nrpages(mapping);`**: 获取地址空间支持的最小 folio 页数 `min_nrpages`。`mapping_min_folio_nrpages(mapping)` 函数 (未提供代码)  很可能返回地址空间配置的最小 folio 大小对应的页数。

```c
	/*
	 * Partway through the readahead operation, we will have added
	 * locked pages to the page cache, but will not yet have submitted
	 * them for I/O.  Adding another page may need to allocate memory,
	 * which can trigger memory reclaim.  Telling the VM we're in
	 * the middle of a filesystem operation will cause it to not
	 * touch file-backed pages, preventing a deadlock.  Most (all?)
	 * filesystems already specify __GFP_NOFS in their mapping's
	 * gfp_mask, but let's be explicit here.
	 */
	unsigned int nofs = memalloc_nofs_save();

	filemap_invalidate_lock_shared(mapping);
	index = mapping_align_index(mapping, index);
```

* **注释块**: 解释了在预读操作过程中，可能需要分配内存，这可能触发内存回收。为了避免死锁，需要告诉 VM 当前正在进行文件系统操作，使其不要触碰文件后备页。虽然大多数文件系统已经在 `mapping->gfp_mask` 中指定了 `__GFP_NOFS`，但这里再次显式地设置。
* **`unsigned int nofs = memalloc_nofs_save();`**: 保存当前的 `NOFS` 状态，并进入 `NOFS` 模式。`memalloc_nofs_save()` 函数 (未提供代码)  很可能设置一个标志，表示当前处于文件系统操作上下文中，限制某些可能导致死锁的内存回收行为。
* **`filemap_invalidate_lock_shared(mapping);`**: 获取地址空间的共享失效锁。`filemap_invalidate_lock_shared` 函数 (未提供代码)  很可能获取一个共享锁，用于保护页缓存的失效操作。
* **`index = mapping_align_index(mapping, index);`**: 对齐起始页索引 `index`。`mapping_align_index(mapping, index)` 函数 (未提供代码)  很可能将 `index` 对齐到 `min_nrpages` 的倍数，确保预读操作以 folio 为单位进行。

```c
	/*
	 * As iterator `i` is aligned to min_nrpages, round_up the
	 * difference between nr_to_read and lookahead_size to mark the
	 * index that only has lookahead or "async_region" to set the
	 * readahead flag.
	 */
	if (lookahead_size <= nr_to_read) {
		unsigned long ra_folio_index;

		ra_folio_index = round_up(readahead_index(ractl) +
					  nr_to_read - lookahead_size,
					  min_nrpages);
		mark = ra_folio_index - index;
	}
	nr_to_read += readahead_index(ractl) - index;
	ractl->_index = index;
```

* **注释块**: 解释了由于迭代器 `i` 对齐到 `min_nrpages`，需要向上取整 `nr_to_read` 和 `lookahead_size` 的差值，以标记只包含 lookahead 或 "async_region" 的索引，用于设置预读标志。
* **`if (lookahead_size <= nr_to_read)`**: 如果 lookahead 大小 `lookahead_size` 小于等于需要读取的页数 `nr_to_read`。
    * **`unsigned long ra_folio_index;`**: 声明变量 `ra_folio_index`，用于存储预读 folio 的索引。
    * **`ra_folio_index = round_up(readahead_index(ractl) + nr_to_read - lookahead_size, min_nrpages);`**: 计算预读 folio 的索引 `ra_folio_index`。将预读起始索引加上 `nr_to_read - lookahead_size`，然后向上取整到 `min_nrpages` 的倍数。`round_up` 宏 (未提供代码)  很可能是向上取整到指定倍数的宏。
    * **`mark = ra_folio_index - index;`**: 计算 `mark` 值，表示从起始索引 `index` 开始，多少个 `min_nrpages` 大小的 folio 之后需要标记为预读页。
* **`nr_to_read += readahead_index(ractl) - index;`**: 调整需要读取的页数 `nr_to_read`。加上 `readahead_index(ractl) - index`，可能是为了补偿之前对起始索引 `index` 的对齐操作。
* **`ractl->_index = index;`**: 设置 `ractl` 中的当前预读索引 `_index` 为对齐后的起始索引 `index`。

```c
	/*
	 * Preallocate as many pages as we will need.
	 */
	while (i < nr_to_read) {
		struct folio *folio = xa_load(&mapping->i_pages, index + i);
		int ret;

		if (folio && !xa_is_value(folio)) {
			/*
			 * Page already present?  Kick off the current batch
			 * of contiguous pages before continuing with the
			 * next batch.  This page may be the one we would
			 * have intended to mark as Readahead, but we don't
			 * have a stable reference to this page, and it's
			 * not worth getting one just for that.
			 */
			read_pages(ractl);
			ractl->_index += min_nrpages;
			i = ractl->_index + ractl->_nr_pages - index;
			continue;
		}

		folio = filemap_alloc_folio(gfp_mask,
					    mapping_min_folio_order(mapping));
		if (!folio)
			break;

		ret = filemap_add_folio(mapping, folio, index + i, gfp_mask);
		if (ret < 0) {
			folio_put(folio);
			if (ret == -ENOMEM)
				break;
			read_pages(ractl);
			ractl->_index += min_nrpages;
			i = ractl->_index + ractl->_nr_pages - index;
			continue;
		}
		if (i == mark)
			folio_set_readahead(folio);
		ractl->_workingset |= folio_test_workingset(folio);
		ractl->_nr_pages += min_nrpages;
		i += min_nrpages;
	}
```

* **注释块**: 解释了预分配尽可能多的页。
* **`while (i < nr_to_read)`**: 循环，直到预分配的页数达到 `nr_to_read`。
    * **`struct folio *folio = xa_load(&mapping->i_pages, index + i);`**: 尝试从地址空间的 `i_pages` 叉树 (radix tree) 中加载索引为 `index + i` 的 folio。`xa_load` 函数 (未提供代码)  很可能用于在叉树中查找 folio，但不增加引用计数。
    * **`int ret;`**: 声明变量 `ret`，用于存储函数返回值。
    * **`if (folio && !xa_is_value(folio))`**: 如果找到 folio 并且不是特殊值 (例如 `XA_VALUE`，用于优化叉树存储)。
        * **注释块**: 解释了如果页已存在，则启动当前批次的连续页，然后再继续下一批次。这个页可能是原本要标记为预读页的，但没有稳定的引用，不值得为了标记而获取引用。
        * `read_pages(ractl);`: **调用 `read_pages` 函数提交当前批次的 I/O 请求。**  `read_pages` 函数 (未提供代码，但根据注释和上下文推断)  很可能是负责将预分配的 folio 提交给底层块设备驱动进行实际的 I/O 读取操作。 **这里是实际 I/O 发起的地方。**
        * `ractl->_index += min_nrpages;`: 更新 `ractl` 中的当前预读索引 `_index`，增加 `min_nrpages`。
        * `i = ractl->_index + ractl->_nr_pages - index;`: 更新循环计数器 `i`，跳过已处理的页。
        * `continue;`: 继续下一次循环迭代。
    * **`folio = filemap_alloc_folio(gfp_mask, mapping_min_folio_order(mapping));`**: 分配新的 folio。`filemap_alloc_folio` 函数 (未提供代码)  很可能使用给定的内存分配标志 `gfp_mask` 和 folio order 分配新的 folio。
    * **`if (!folio) break;`**: 如果 folio 分配失败，则跳出循环。
    * **`ret = filemap_add_folio(mapping, folio, index + i, gfp_mask);`**: 将新分配的 folio 添加到地址空间的 `i_pages` 叉树中。`filemap_add_folio` 函数 (未提供代码)  很可能将 folio 添加到叉树，并增加 folio 的引用计数。
    * **`if (ret < 0)`**: 如果 `filemap_add_folio` 返回错误。
        * `folio_put(folio);`: 释放 folio 的引用计数。
        * `if (ret == -ENOMEM) break;`: 如果错误是 `-ENOMEM` (内存不足)，则跳出循环。
        * `read_pages(ractl);`: 调用 `read_pages` 函数提交当前批次的 I/O 请求。
        * `ractl->_index += min_nrpages;`: 更新 `ractl` 中的当前预读索引 `_index`，增加 `min_nrpages`。
        * `i = ractl->_index + ractl->_nr_pages - index;`: 更新循环计数器 `i`，跳过已处理的页。
        * `continue;`: 继续下一次循环迭代。
    * **`if (i == mark) folio_set_readahead(folio);`**: 如果当前循环计数器 `i` 等于 `mark` 值，则将当前 folio 标记为预读页。`folio_set_readahead` 函数 (未提供代码)  很可能设置 folio 的某个标志位，表示这是一个预读页。
    * **`ractl->_workingset |= folio_test_workingset(folio);`**: 更新 `ractl` 的工作集状态。`folio_test_workingset(folio)` 函数 (未提供代码)  很可能检查 folio 是否属于工作集，并将结果与 `ractl->_workingset` 进行或运算。
    * **`ractl->_nr_pages += min_nrpages;`**: 增加 `ractl` 中已处理的页数计数器 `_nr_pages`，增加 `min_nrpages`。
    * **`i += min_nrpages;`**: 循环计数器 `i` 增加 `min_nrpages`。

```c
	/*
	 * Now start the IO.  We ignore I/O errors - if the folio is not
	 * uptodate then the caller will launch read_folio again, and
	 * will then handle the error.
	 */
	read_pages(ractl);
	filemap_invalidate_unlock_shared(mapping);
	memalloc_nofs_restore(nofs);
}
EXPORT_SYMBOL_GPL(page_cache_ra_unbounded);
```

* **注释块**: 解释了现在启动 I/O。忽略 I/O 错误，如果 folio 不是 `uptodate`，则调用者会再次启动 `read_folio`，并处理错误。
* **`read_pages(ractl);`**: **再次调用 `read_pages` 函数提交剩余的 I/O 请求。**  在循环结束后，可能还有一些预分配的 folio 没有提交 I/O，这里再次调用 `read_pages` 确保所有预分配的 folio 都被提交。
* **`filemap_invalidate_unlock_shared(mapping);`**: 释放地址空间的共享失效锁。
* **`memalloc_nofs_restore(nofs);`**: 恢复之前的 `NOFS` 状态。
* **`EXPORT_SYMBOL_GPL(page_cache_ra_unbounded);`**: 导出 `page_cache_ra_unbounded` 函数为内核符号，并声明使用 GPL 协议。

**总结 `page_cache_ra_unbounded`:**

`page_cache_ra_unbounded` 函数是实际执行无界预读的核心函数。它的主要步骤包括：

1.  **初始化和准备工作**: 设置 `NOFS` 状态，获取共享失效锁，对齐起始索引。
2.  **预分配 folio**: 循环预分配指定数量的 folio，并将它们添加到地址空间的 `i_pages` 叉树中。如果 folio 已经存在，则跳过分配。
3.  **标记预读页**: 根据 `lookahead_size` 标记一部分 folio 为预读页。
4.  **提交 I/O**:  在循环过程中和循环结束后，都调用 `read_pages` 函数将预分配的 folio 提交给底层块设备驱动进行实际的 I/O 读取。
5.  **清理工作**: 释放共享失效锁，恢复 `NOFS` 状态。

**函数调用关系和 readahead 流程总结**

从 `filemap_get_pages` 到 `page_cache_ra_unbounded` 的调用链，以及这些函数之间的协作，可以总结出同步预读的流程如下：

1.  **`filemap_get_pages`**:  在需要读取数据时被调用。它首先尝试从页缓存获取 folio，如果未命中，则调用 `page_cache_sync_readahead` 触发同步预读。
2.  **`page_cache_sync_readahead`**:  初始化 `readahead_control` 结构体，并调用 `page_cache_sync_ra` 执行核心预读逻辑。
3.  **`page_cache_sync_ra`**:  根据文件访问模式、预读状态和缓存命中情况，决定预读策略和大小。它可能调用 `force_page_cache_ra` (强制预读) 或 `page_cache_ra_order` (顺序预读，代码未提供)。 在强制预读的情况下，会调用 `force_page_cache_ra`.
4.  **`force_page_cache_ra`**:  将预读请求分块为 2MB 单位，并循环调用 `do_page_cache_ra` 来执行每个分块的预读。
5.  **`do_page_cache_ra`**:  限制预读范围，确保不超过文件末尾，然后调用 `page_cache_ra_unbounded` 执行实际的无界预读。
6.  **`page_cache_ra_unbounded`**:  预分配 folio，添加到页缓存，标记预读页，并**通过 `read_pages` 函数提交 I/O 请求**。 `read_pages` 函数是实际发起块设备 I/O 操作的关键，但代码未提供。

**关键点:**

*   **`read_pages` 函数**:  虽然代码未提供，但它是整个预读流程中**真正发起 I/O 读取操作**的函数。它很可能与地址空间 `mapping->a_ops` 中的 `readpage` 或 `readpages` 操作关联，最终调用底层的块设备驱动来读取数据。
*   **`readahead_control` 结构体**:  用于管理预读操作的上下文信息，在各个预读函数之间传递。
*   **分层预读策略**:  从 `page_cache_sync_readahead` 到 `page_cache_ra_unbounded`，体现了分层的预读策略，根据不同的情况选择不同的预读方式和参数，以优化性能。
*   **内存管理**:  强制预读分块为 2MB 单位，以及预分配 folio 的机制，都是为了控制预读操作的内存占用，避免一次性锁定过多内存。

希望这个详细的解析能够帮助您更深入地理解 Linux 内核的同步预读机制。如果您还有其他问题，或者想继续深入研究其他函数，请随时提出。

