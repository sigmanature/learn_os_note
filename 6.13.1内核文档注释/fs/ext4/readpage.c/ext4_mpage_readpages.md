 **相关函数**
 * 
 好的，让我们来逐行分析 `ext4_mpage_readpages` 函数。这个函数在 ext4 文件系统中负责读取多个页面的数据，通常用于预读（readahead）操作，以提高文件读取性能。

```c
int ext4_mpage_readpages(struct inode *inode,
		struct readahead_control *rac, struct folio *folio)
{
```
**函数签名:**
- `int ext4_mpage_readpages(...)`:  定义了一个名为 `ext4_mpage_readpages` 的函数，返回类型为 `int`，通常 0 表示成功，负值表示错误。
- `struct inode *inode`:  指向 `inode` 结构的指针。`inode` 结构体包含了文件的元数据信息，例如文件大小、权限、块映射等。这个参数指定了要读取数据的文件。
- `struct readahead_control *rac`: 指向 `readahead_control` 结构的指针。如果进行预读操作，这个参数会提供预读控制信息，例如预读窗口大小等。如果 `rac` 为 `NULL`，则表示只读取当前 `folio` 页。
- `struct folio *folio`: 指向 `folio` 结构的指针。`folio` 是 Linux 内核中用于管理内存页的新结构，类似于 `page` 结构，但更加通用和高效。这个参数指定了要读取数据的目标内存页。

```c
	struct bio *bio = NULL;
	sector_t last_block_in_bio = 0;
```
**变量声明:**
- `struct bio *bio = NULL;`: 声明一个指向 `bio` 结构的指针 `bio` 并初始化为 `NULL`。`bio` (Block I/O) 结构体是 Linux 内核中表示块设备 I/O 操作的基本结构。这个变量将用于构建和提交读取请求。
- `sector_t last_block_in_bio = 0;`: 声明一个 `sector_t` 类型的变量 `last_block_in_bio` 并初始化为 0。`sector_t` 是扇区号的类型。这个变量用于跟踪当前 `bio` 中最后一个块的扇区号，用于判断是否可以合并后续的块到同一个 `bio` 中。

```c
	const unsigned blkbits = inode->i_blkbits;
	const unsigned blocks_per_page = PAGE_SIZE >> blkbits;
	const unsigned blocksize = 1 << blkbits;
	sector_t next_block;
	sector_t block_in_file;
	sector_t last_block;
	sector_t last_block_in_file;
	sector_t first_block;
	unsigned page_block;
	struct block_device *bdev = inode->i_sb->s_bdev;
	int length;
	unsigned relative_block = 0;
	struct ext4_map_blocks map;
	unsigned int nr_pages = rac ? readahead_count(rac) : 1;
```
**变量声明 (续):**
- `const unsigned blkbits = inode->i_blkbits;`: 获取文件系统块大小的位数。`inode->i_blkbits` 存储了文件系统块大小的以 2 为底的对数。例如，如果块大小是 4KB，则 `blkbits` 为 12 (2<sup>12</sup> = 4096)。`const` 表示这个值在函数执行期间不会改变。
- `const unsigned blocks_per_page = PAGE_SIZE >> blkbits;`: 计算每个内存页包含的块的数量。`PAGE_SIZE` 是系统页大小（通常是 4KB 或更大）。右移 `blkbits` 位相当于除以 2<sup>blkbits</sup>，即除以块大小。
- `const unsigned blocksize = 1 << blkbits;`: 计算块大小，通过左移 `blkbits` 位，即 2<sup>blkbits</sup>。
- `sector_t next_block;`: 声明 `sector_t` 类型的变量 `next_block`，用于跟踪下一个要读取的逻辑块号。
- `sector_t block_in_file;`: 声明 `sector_t` 类型的变量 `block_in_file`，表示当前正在处理的文件内的逻辑块号。
- `sector_t last_block;`: 声明 `sector_t` 类型的变量 `last_block`，表示当前预读操作的最后一个逻辑块号。
- `sector_t last_block_in_file;`: 声明 `sector_t` 类型的变量 `last_block_in_file`，表示文件在文件系统中的最后一个逻辑块号。
- `sector_t first_block;`: 声明 `sector_t` 类型的变量 `first_block`，用于存储当前连续块区域的起始物理块号。
- `unsigned page_block;`: 声明 `unsigned int` 类型的变量 `page_block`，用于跟踪当前页内已经处理的块的数量。
- `struct block_device *bdev = inode->i_sb->s_bdev;`: 获取块设备结构体指针 `bdev`。`inode->i_sb` 指向超级块结构体，`inode->i_sb->s_bdev` 指向底层的块设备结构体，代表文件系统所在的磁盘设备。
- `int length;`: 声明 `int` 类型的变量 `length`，用于存储要读取的数据长度。
- `unsigned relative_block = 0;`: 声明 `unsigned int` 类型的变量 `relative_block` 并初始化为 0，用于跟踪当前块在 `ext4_map_blocks` 返回的连续块区域内的相对位置。
- `struct ext4_map_blocks map;`: 声明 `ext4_map_blocks` 类型的结构体变量 `map`。这个结构体用于存储块映射信息，包括逻辑块号、物理块号、长度和标志。
- `unsigned int nr_pages = rac ? readahead_count(rac) : 1;`:  确定要读取的页数。如果 `rac` 非空（表示进行预读），则调用 `readahead_count(rac)` 获取预读的页数；否则，只读取 1 页（当前 `folio` 页）。

```c
	map.m_pblk = 0;
	map.m_lblk = 0;
	map.m_len = 0;
	map.m_flags = 0;
```
**初始化 `map` 结构体:**
- `map.m_pblk = 0;`: 初始化 `map` 结构体的物理块号 `m_pblk` 为 0。
- `map.m_lblk = 0;`: 初始化 `map` 结构体的逻辑块号 `m_lblk` 为 0。
- `map.m_len = 0;`: 初始化 `map` 结构体的长度 `m_len` 为 0。
- `map.m_flags = 0;`: 初始化 `map` 结构体的标志 `m_flags` 为 0。这些初始化操作清空了 `map` 结构体，为后续的块映射操作做准备。

```c
	for (; nr_pages; nr_pages--) {
```
**主循环开始:**
- `for (; nr_pages; nr_pages--)`:  这是一个 `for` 循环，循环条件是 `nr_pages` 大于 0。每次循环迭代后，`nr_pages` 减 1。这个循环会执行 `nr_pages` 次，每次处理一个内存页。

```c
		int fully_mapped = 1;
		unsigned first_hole = blocks_per_page;
```
**循环内部变量初始化:**
- `int fully_mapped = 1;`: 初始化 `fully_mapped` 变量为 1（真）。这个变量用于标记当前页是否所有逻辑块都成功映射到物理块。初始假设是完全映射的，如果在后续处理中发现有未映射的块（空洞），则会被设置为 0（假）。
- `unsigned first_hole = blocks_per_page;`: 初始化 `first_hole` 变量为 `blocks_per_page`。这个变量用于记录当前页中第一个未映射块（空洞）的起始块索引。如果整个页都是连续映射的，则 `first_hole` 保持为 `blocks_per_page`。

```c
		if (rac)
			folio = readahead_folio(rac);
		prefetchw(&folio->flags);
```
**预读页获取和预取:**
- `if (rac)`: 检查 `rac` 是否非空，即是否进行预读操作。
- `folio = readahead_folio(rac);`: 如果 `rac` 非空，则调用 `readahead_folio(rac)` 获取下一个要预读的 `folio`。这个函数会根据预读策略选择下一个要读取的页。
- `prefetchw(&folio->flags);`:  使用 `prefetchw` 指令预取 `folio->flags` 的数据到缓存。这是一种性能优化手段，提前将页标志加载到缓存，以减少后续访问延迟。

```c
		if (folio_buffers(folio))
			goto confused;
```
**检查 folio 是否已经关联 buffer:**
- `if (folio_buffers(folio))`: 检查 `folio` 是否已经关联了 buffer (缓冲区)。`folio_buffers(folio)` 检查 `folio` 是否已经通过 `create_folio_buffers` 或类似函数分配了 buffer head。
- `goto confused;`: 如果 `folio_buffers(folio)` 返回真（非零），表示 `folio` 已经关联了 buffer，这在 `mpage_readpages` 的上下文中是不应该发生的（因为这个函数期望处理的是新的、未初始化的 folio）。在这种“困惑”的情况下，跳转到 `confused` 标签进行错误处理。

```c
		block_in_file = next_block =
			(sector_t)folio->index << (PAGE_SHIFT - blkbits);
		last_block = block_in_file + nr_pages * blocks_per_page;
		last_block_in_file = (ext4_readpage_limit(inode) +
				      blocksize - 1) >> blkbits;
		if (last_block > last_block_in_file)
			last_block = last_block_in_file;
		page_block = 0;
```
**计算块范围:**
- `block_in_file = next_block = (sector_t)folio->index << (PAGE_SHIFT - blkbits);`: 计算当前 `folio` 在文件中的起始逻辑块号。`folio->index` 是 `folio` 的索引号（页号）。`PAGE_SHIFT - blkbits` 计算出页大小和块大小的位数差，左移操作相当于乘以 2<sup>(PAGE_SHIFT - blkbits)</sup>，将页索引转换为文件内的起始块号。`next_block` 也被初始化为这个值，用于后续迭代。举个例子,folio->index为0的话逻辑块号就是0,index为1的话逻辑块号就是1,映射关系很明确。
- `last_block = block_in_file + nr_pages * blocks_per_page;`: 计算本次预读操作的最后一个逻辑块号的**理论值**。假设要预读 `nr_pages` 页，每页 `blocks_per_page` 个块，则总共要读取 `nr_pages * blocks_per_page` 个块。
- `last_block_in_file = (ext4_readpage_limit(inode) + blocksize - 1) >> blkbits;`: 计算文件在文件系统中的最后一个逻辑块号。`ext4_readpage_limit(inode)` 返回文件读取的上限字节数。加上 `blocksize - 1` 并右移 `blkbits` 位，是为了向上取整，确保包含文件的最后一个块。
- `if (last_block > last_block_in_file) last_block = last_block_in_file;`: 调整 `last_block`，确保预读范围不超过文件的实际大小。如果计算出的 `last_block` 超出了文件的最后一个块，则将其设置为文件的最后一个块。
- `page_block = 0;`: 初始化 `page_block` 为 0，表示当前页内已处理的块数为 0。

```c
		/*
		 * Map blocks using the previous result first.
		 */
		if ((map.m_flags & EXT4_MAP_MAPPED) &&
		    block_in_file > map.m_lblk &&
		    block_in_file < (map.m_lblk + map.m_len)) {
			unsigned map_offset = block_in_file - map.m_lblk;
			unsigned last = map.m_len - map_offset;

			first_block = map.m_pblk + map_offset;
			for (relative_block = 0; ; relative_block++) {
				if (relative_block == last) {
					/* needed? */
					map.m_flags &= ~EXT4_MAP_MAPPED;
					break;
				}
				if (page_block == blocks_per_page)
					break;
				page_block++;
				block_in_file++;
			}
		}
```
**尝试使用上次块映射结果:**
- `if ((map.m_flags & EXT4_MAP_MAPPED) && ...)`:  检查是否上次 `ext4_map_blocks` 调用成功映射了块 (`EXT4_MAP_MAPPED` 标志被设置)，并且当前要读取的块 `block_in_file` 落在上次映射的逻辑块范围内 (`block_in_file > map.m_lblk && block_in_file < (map.m_lblk + map.m_len)`)。如果是，则尝试利用上次的映射结果，避免重复调用 `ext4_map_blocks`。
- `unsigned map_offset = block_in_file - map.m_lblk;`: 计算当前块在上次映射区域内的偏移量。
- `unsigned last = map.m_len - map_offset;`: 计算上次映射区域中，从当前块开始到结束的剩余长度。
- `first_block = map.m_pblk + map_offset;`: 计算当前块的物理块号，通过上次映射的起始物理块号 `map.m_pblk` 加上偏移量 `map_offset` 得到。
- `for (relative_block = 0; ; relative_block++)`:  一个循环，遍历上次映射区域中与当前页相关的块。
    - `if (relative_block == last)`: 如果已经遍历完上次映射区域的剩余部分，则清除 `EXT4_MAP_MAPPED` 标志，表示上次映射结果不再有效，并跳出循环。
    - `if (page_block == blocks_per_page)`: 如果当前页已经填满，则跳出循环。
    - `page_block++; block_in_file++;`: 递增页内块计数器 `page_block` 和文件内块号 `block_in_file`，处理下一个块。

```c
		/*
		 * Then do more ext4_map_blocks() calls until we are
		 * done with this folio.
		 */
        /*如前所述,page_block表示的是当前页内已经处理的块数量 而block_in_file表示当前已经处理的全局文件块数量*/
		while (page_block < blocks_per_page) {
			if (block_in_file < last_block) {
				map.m_lblk = block_in_file;
				map.m_len = last_block - block_in_file;

				if (ext4_map_blocks(NULL, inode, &map, 0) < 0) {
				set_error_page:
					folio_zero_segment(folio, 0,
							  folio_size(folio));
					folio_unlock(folio);
					goto next_page;
				}
			}
			if ((map.m_flags & EXT4_MAP_MAPPED) == 0) {
				fully_mapped = 0;
				if (first_hole == blocks_per_page)
					first_hole = page_block;
				page_block++;
				block_in_file++;
				continue;
			}
			if (first_hole != blocks_per_page)
				goto confused;		/* hole -> non-hole */

			/* Contiguous blocks? */
			if (!page_block)
				first_block = map.m_pblk;
			else if (first_block + page_block != map.m_pblk)
				goto confused;
			for (relative_block = 0; ; relative_block++) {
				if (relative_block == map.m_len) {
					/* needed? */
					map.m_flags &= ~EXT4_MAP_MAPPED;
					break;
				} else if (page_block == blocks_per_page)
					break;
				page_block++;
				block_in_file++;
			}
		}
```
**循环调用 `ext4_map_blocks` 进行块映射:**
- `while (page_block < blocks_per_page)`:  只要当前页还没有填满，就继续循环。
- `if (block_in_file < last_block)`: 检查当前要读取的块是否在预读范围内 (`last_block`)。
    - `map.m_lblk = block_in_file;`: 设置 `map` 结构体的逻辑块号为当前块 `block_in_file`。
    - `map.m_len = last_block - block_in_file;`: 设置 `map` 结构体的长度为剩余预读范围。
    - `if (ext4_map_blocks(NULL, inode, &map, 0) < 0)`: 调用 `ext4_map_blocks` 函数进行块映射。
        - `NULL`:  第一个参数通常是 transaction handle，这里为 `NULL` 表示不在事务上下文中。
        - `inode`:  文件 inode。
        - `&map`:  指向 `map` 结构体的指针，用于传递逻辑块号和接收物理块号等映射信息。
        - `0`:  标志位，这里为 0 表示默认行为。
        - 如果 `ext4_map_blocks` 返回值小于 0，表示映射失败。跳转到 `set_error_page` 标签进行错误处理。
    - `set_error_page:` 标签:
        - `folio_zero_segment(folio, 0, folio_size(folio));`: 将整个 `folio` 页清零。
        - `folio_unlock(folio);`: 解锁 `folio` 页。
        - `goto next_page;`: 跳转到 `next_page` 标签，处理下一个页。
- `if ((map.m_flags & EXT4_MAP_MAPPED) == 0)`: 检查 `ext4_map_blocks` 是否成功映射了块（`EXT4_MAP_MAPPED` 标志是否被设置）。如果没有映射成功，表示遇到了空洞（hole）。
    - `fully_mapped = 0;`: 设置 `fully_mapped` 为 0，表示当前页不是完全映射的。
    - `if (first_hole == blocks_per_page) first_hole = page_block;`: 如果 `first_hole` 仍然是初始值 `blocks_per_page`，说明这是第一次遇到空洞，记录空洞的起始块索引为 `page_block`。
    - `page_block++; block_in_file++; continue;`: 递增 `page_block` 和 `block_in_file`，继续处理下一个块，并跳过后续的连续块处理逻辑 (`continue` 语句)。
- `if (first_hole != blocks_per_page) goto confused;`: 如果 `first_hole` 不等于 `blocks_per_page`，并且当前块是映射成功的，这意味着在同一个页中，先出现了空洞，然后又出现了映射块，这种情况在 `mpage_readpages` 的逻辑中是不应该发生的（期望空洞是连续的在页的末尾）。跳转到 `confused` 标签进行错误处理。
- **连续块处理:** (如果代码执行到这里，说明当前块是映射成功的，并且之前没有遇到空洞，或者当前块是页内的第一个块)
    - `if (!page_block) first_block = map.m_pblk;`: 如果 `page_block` 为 0，表示这是页内的第一个块，将 `first_block` 设置为当前映射的物理块号 `map.m_pblk`。
    - `else if (first_block + page_block != map.m_pblk) goto confused;`: 如果 `page_block` 不为 0，检查当前映射的物理块号 `map.m_pblk` 是否与预期的连续物理块号 `first_block + page_block` 相符。如果不符，表示块不连续，跳转到 `confused` 标签进行错误处理。
    - `for (relative_block = 0; ; relative_block++)`:  循环处理当前 `ext4_map_blocks` 返回的连续块区域。
        - `if (relative_block == map.m_len)`: 如果已经遍历完当前映射区域的所有块，则清除 `EXT4_MAP_MAPPED` 标志，表示当前映射结果不再有效，并跳出循环。
        - `else if (page_block == blocks_per_page)`: 如果当前页已经填满，则跳出循环。
        - `page_block++; block_in_file++;`: 递增 `page_block` 和 `block_in_file`，处理下一个块。

```c
		if (first_hole != blocks_per_page) {
			folio_zero_segment(folio, first_hole << blkbits,
					  folio_size(folio));
			if (first_hole == 0) {
				if (ext4_need_verity(inode, folio->index) &&
				    !fsverity_verify_folio(folio))
					goto set_error_page;
				folio_end_read(folio, true);
				continue;
			}
		} else if (fully_mapped) {
			folio_set_mappedtodisk(folio);
		}
```
**处理页内空洞和标记页状态:**
- `if (first_hole != blocks_per_page)`: 如果 `first_hole` 不等于 `blocks_per_page`，表示当前页存在空洞。
    - `folio_zero_segment(folio, first_hole << blkbits, folio_size(folio));`: 将 `folio` 页中从空洞起始位置 (`first_hole << blkbits`) 到页末尾的部分清零。
    - `if (first_hole == 0)`: 如果空洞从页的起始位置开始 (`first_hole == 0`)，表示整个页都是空洞。
        - `if (ext4_need_verity(inode, folio->index) && !fsverity_verify_folio(folio)) goto set_error_page;`: 如果文件系统需要 verity 校验 (`ext4_need_verity`)，并且 verity 校验失败 (`!fsverity_verify_folio`)，则跳转到 `set_error_page` 标签进行错误处理。verity 是一种用于数据完整性校验的机制。
        - `folio_end_read(folio, true);`: 标记 `folio` 页读取完成，状态为 uptodate (true)。即使是空洞页，也需要标记为读取完成，避免后续重复读取。
        - `continue;`: 继续处理下一个页。
- `else if (fully_mapped)`: 如果 `first_hole` 等于 `blocks_per_page`，表示整个页都是映射的，并且 `fully_mapped` 为真，表示页内所有块都成功映射。
    - `folio_set_mappedtodisk(folio);`: 设置 `folio` 页的 `mappedtodisk` 标志，表示页内容与磁盘上的数据一致。

```c
		/*
		 * This folio will go to BIO.  Do we need to send this
		 * BIO off first?
		 */
		if (bio && (last_block_in_bio != first_block - 1 ||
			    !fscrypt_mergeable_bio(bio, inode, next_block))) {
		submit_and_realloc:
			submit_bio(bio);
			bio = NULL;
		}
		if (bio == NULL) {
			/*
			 * bio_alloc will _always_ be able to allocate a bio if
			 * __GFP_DIRECT_RECLAIM is set, see bio_alloc_bioset().
			 */
			bio = bio_alloc(bdev, bio_max_segs(nr_pages),
					REQ_OP_READ, GFP_KERNEL);
			fscrypt_set_bio_crypt_ctx(bio, inode, next_block,
						  GFP_KERNEL);
			ext4_set_bio_post_read_ctx(bio, inode, folio->index);
			bio->bi_iter.bi_sector = first_block << (blkbits - 9);
			bio->bi_end_io = mpage_end_io;
			if (rac)
				bio->bi_opf |= REQ_RAHEAD;
		}
```
**BIO 管理和分配:**
- `if (bio && (last_block_in_bio != first_block - 1 || !fscrypt_mergeable_bio(bio, inode, next_block)))`: 检查是否需要提交当前的 `bio`。
    - `bio`: 检查 `bio` 是否非空，即当前是否已经有一个 `bio` 正在构建。
    - `last_block_in_bio != first_block - 1`: 检查当前页的起始物理块号 `first_block` 是否与上一个 `bio` 的最后一个块号 `last_block_in_bio` 连续。如果不连续，则需要提交当前的 `bio`。
    - `!fscrypt_mergeable_bio(bio, inode, next_block)`: 检查当前的 `folio` 是否可以合并到现有的 `bio` 中，考虑到文件系统加密 (fscrypt)。如果不能合并，则需要提交当前的 `bio`。
    - 如果上述条件之一为真，则跳转到 `submit_and_realloc` 标签。
- `submit_and_realloc:` 标签:
    - `submit_bio(bio);`: 提交当前的 `bio`，将读取请求发送到块设备层。
    - `bio = NULL;`: 将 `bio` 指针设置为 `NULL`，表示当前的 `bio` 已经提交，需要重新分配。
- `if (bio == NULL)`: 如果 `bio` 为空，表示需要分配一个新的 `bio`。
    - `bio = bio_alloc(bdev, bio_max_segs(nr_pages), REQ_OP_READ, GFP_KERNEL);`: 分配一个新的 `bio` 结构体。
        - `bdev`: 块设备结构体指针。
        - `bio_max_segs(nr_pages)`: 计算 `bio` 可以容纳的最大段数，根据要读取的页数 `nr_pages` 确定。
        - `REQ_OP_READ`: 指定 I/O 操作类型为读操作。
        - `GFP_KERNEL`:  内存分配标志，`GFP_KERNEL` 表示在内核上下文中分配内存，可能会阻塞。
    - `fscrypt_set_bio_crypt_ctx(bio, inode, next_block, GFP_KERNEL);`: 如果文件系统使用了 fscrypt 加密，则设置 `bio` 的加密上下文。
    - `ext4_set_bio_post_read_ctx(bio, inode, folio->index);`: 设置 ext4 特有的 `bio` 后处理上下文，用于在 I/O 完成后执行特定操作。
    - `bio->bi_iter.bi_sector = first_block << (blkbits - 9);`: 设置 `bio` 的起始扇区号。`first_block << (blkbits - 9)` 将物理块号转换为扇区号（假设扇区大小为 512 字节，即 2<sup>9</sup> 字节）。
    - `bio->bi_end_io = mpage_end_io;`: 设置 `bio` 的 I/O 完成回调函数为 `mpage_end_io`，当 I/O 操作完成时，内核会调用这个函数进行后续处理。
    - `if (rac) bio->bi_opf |= REQ_RAHEAD;`: 如果 `rac` 非空（进行预读），则设置 `bio` 的操作标志 `bi_opf` 包含 `REQ_RAHEAD`，表示这是一个预读请求。

```c
		length = first_hole << blkbits;
		if (!bio_add_folio(bio, folio, length, 0))
			goto submit_and_realloc;
```
**向 BIO 添加 folio:**
- `length = first_hole << blkbits;`: 计算要添加到 `bio` 的数据长度，即从页起始位置到第一个空洞起始位置的数据长度。
- `if (!bio_add_folio(bio, folio, length, 0)) goto submit_and_realloc;`: 将当前的 `folio` 页添加到 `bio` 中。
    - `bio_add_folio(bio, folio, length, 0)`: 尝试将 `folio` 的前 `length` 字节添加到 `bio` 中。
    - 如果 `bio_add_folio` 返回 0，表示添加失败（例如，`bio` 已经满了），则跳转到 `submit_and_realloc` 标签，提交当前的 `bio` 并重新分配一个新的 `bio`。

```c
		if (((map.m_flags & EXT4_MAP_BOUNDARY) &&
		     (relative_block == map.m_len)) ||
		    (first_hole != blocks_per_page)) {
			submit_bio(bio);
			bio = NULL;
		} else
			last_block_in_bio = first_block + blocks_per_page - 1;
		continue;
```
**BIO 提交条件和更新 `last_block_in_bio`:**
- `if (((map.m_flags & EXT4_MAP_BOUNDARY) && (relative_block == map.m_len)) || (first_hole != blocks_per_page))`: 检查是否需要提交当前的 `bio`。
    - `(map.m_flags & EXT4_MAP_BOUNDARY) && (relative_block == map.m_len)`: 如果 `ext4_map_blocks` 返回的映射区域有边界标志 (`EXT4_MAP_BOUNDARY`)，并且已经处理完这个区域的所有块 (`relative_block == map.m_len`)，则需要提交 `bio`。边界标志可能表示映射区域的结束，例如文件结尾或元数据区域的边界。
    - `(first_hole != blocks_per_page)`: 如果当前页存在空洞 (`first_hole != blocks_per_page`)，也需要提交 `bio`。因为空洞标志着当前连续读取的结束。
    - 如果上述条件之一为真，则执行以下操作：
        - `submit_bio(bio);`: 提交当前的 `bio`。
        - `bio = NULL;`: 将 `bio` 指针设置为 `NULL`。
- `else last_block_in_bio = first_block + blocks_per_page - 1;`: 如果不需要提交 `bio`，则更新 `last_block_in_bio` 为当前页的最后一个块号。这用于在下一次循环迭代中判断是否可以合并后续的块到同一个 `bio` 中。
- `continue;`: 继续处理下一个页。

```c
	confused:
		if (bio) {
			submit_bio(bio);
			bio = NULL;
		}
		if (!folio_test_uptodate(folio))
			block_read_full_folio(folio, ext4_get_block);
		else
			folio_unlock(folio);
next_page:
		; /* A label shall be followed by a statement until C23 */
	}
```
**`confused` 标签和 `next_page` 标签:**
- `confused:` 标签:  错误处理逻辑。当代码执行到 `confused` 标签时，表示遇到了不期望的情况，需要进行清理和错误处理。
    - `if (bio) { submit_bio(bio); bio = NULL; }`: 如果当前有 `bio` 正在构建，则提交它并释放 `bio` 结构体。
    - `if (!folio_test_uptodate(folio)) block_read_full_folio(folio, ext4_get_block);`: 如果 `folio` 页没有被标记为 uptodate (读取完成)，则调用 `block_read_full_folio` 函数进行同步读取。`block_read_full_folio` 会阻塞当前进程，直到整个 `folio` 页的数据从磁盘读取完成。`ext4_get_block` 是一个用于获取块映射的回调函数。
    - `else folio_unlock(folio);`: 如果 `folio` 页已经被标记为 uptodate，则只需要解锁 `folio` 页即可。
- `next_page:` 标签:  循环的下一个迭代点。
    - `; /* A label shall be followed by a statement until C23 */`:  这是一个空语句，在 C23 标准之前，标签后面必须跟一个语句。这里使用空语句来满足语法要求。

```c
	if (bio)
		submit_bio(bio);
	return 0;
}
```
**函数结束处理:**
- `if (bio) submit_bio(bio);`: 在主循环结束后，如果 `bio` 仍然非空，表示还有未提交的 `bio`，需要提交它。这通常发生在循环处理完所有页后，最后一个 `bio` 还没有被提交。
- `return 0;`: 函数返回 0，表示函数执行成功。

**总结:**

`ext4_mpage_readpages` 函数的主要功能是读取多个连续的页面，通常用于预读操作。它通过循环处理每个页面，并使用 `ext4_map_blocks` 函数将逻辑块号映射到物理块号。函数会尝试合并连续的块到同一个 `bio` 结构体中，以提高 I/O 效率。在处理过程中，函数会处理文件空洞、边界情况和错误情况。对于每个页面，函数会构建 `bio` 结构体，并将其提交到块设备层进行实际的 I/O 操作。函数还考虑了文件系统加密和 verity 校验等特性。

希望这个逐行分析对您有所帮助! 如果您有任何其他问题，请随时提出。

