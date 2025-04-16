处理常规（非原子）写入的块查找/预留。
**相关函数**
* [f2fs_write_begin](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/data.c/f2fs_write_begin.md)
* [f2fs_convert_inline_page](https://github.com/sigmanature/learn_os_note/blob/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/inline.c/f2fs_convert_inline_page.md)
```c
// prepare_write_begin: 用于普通文件的 f2fs_write_begin 辅助函数
static int prepare_write_begin(struct f2fs_sb_info *sbi,
			struct folio *folio, loff_t pos, unsigned int len,
			block_t *blk_addr, bool *node_changed)/*最诡异的是node_changed这个变量指示是否要对文件系统做平衡操作?*/
{
	struct inode *inode = folio->mapping->host; // 获取 inode
	pgoff_t index = folio->index; // 页索引
	struct dnode_of_data dn; // 用于保存数据节点信息（块地址、节点页）的结构
	struct page *ipage = NULL; // 包含 inode 节点块（元数据）的页
	bool locked = false; // 标志：是否持有 f2fs_map_lock？
	// 块分配行为标志（例如，AIO 上下文提示）
	int flag = F2FS_GET_BLOCK_PRE_AIO;
	int err = 0;

	/*
	 * 如果正在写入整页并且我们已经预分配了所有块，
	 * 那么现在无需获取块地址。
	 * 实际的块分配将在稍后的检查点/回写期间进行。
	 * 这依赖于由 f2fs_preallocate_blocks 设置的 FI_PREALLOCATED_ALL 标志。
	 */
	if (len == PAGE_SIZE && is_inode_flag_set(inode, FI_PREALLOCATED_ALL)) {
		*blk_addr = NEW_ADDR; // 标记它在逻辑上是新的/预分配的
		*node_changed = true; // 假设元数据会改变
		return 0; // 优化成功
	}


	/* f2fs_lock_op 避免了写入 CP 和 convert_inline_page 之间的竞争 */
	// 在特定条件下获取 F2FS 映射锁以保护元数据操作：
	// 1. 如果 inode 当前有内联数据（可能需要转换）。
	// 2. 如果写入超出当前文件末尾（可能需要分配）。
	if (f2fs_has_inline_data(inode)) {
		if (pos + len > MAX_INLINE_DATA(inode))
			flag = F2FS_GET_BLOCK_DEFAULT;
		f2fs_map_lock(sbi, flag); // 获取锁
		locked = true;
	} else if ((pos & PAGE_MASK) >= i_size_read(inode)) { // 写入超出 EOF
		f2fs_map_lock(sbi, flag); // 获取锁
		locked = true;
	}

restart: // 如果需要，在获取锁后重试的标签
	/* 检查内联数据 */
	// 获取包含 inode 直接/间接指针的节点页。
	ipage = f2fs_get_node_page(sbi, inode->i_ino);
	if (IS_ERR(ipage)) { // 获取 inode 节点页失败
		err = PTR_ERR(ipage);
		goto unlock_out;
	}

	// 使用 inode 及其节点页初始化 dnode_of_data 结构
	set_new_dnode(&dn, inode, ipage, ipage, 0); // 此处 ipage 既是 nid_page 也是 node_page

	// --- 处理内联数据 ---
	if (f2fs_has_inline_data(inode)) {
		// 如果写入完全在内联数据区域内：
		if (pos + len <= MAX_INLINE_DATA(inode)) {
			// 直接从 inode 的节点页读取现有内联数据到 folio。
			f2fs_do_read_inline_data(folio, ipage);
			set_inode_flag(inode, FI_DATA_EXIST); // 标记数据存在（内联）
			// 如果 inode 有链接，则特殊处理页面私有标志
			if (inode->i_nlink)
				set_page_private_inline(ipage);
			// 不需要块地址（blk_addr 隐式保持 NULL_ADDR）。
			// 此处不需要块分配。
			goto out; // 成功，跳过块查找
		}
		// 写入超出了内联数据容量，需要转换为普通块。
		// 此函数分配第一个数据块并将内联数据移动到其中。
		err = f2fs_convert_inline_page(&dn, folio_page(folio, 0));
		// 如果转换发生，dn.data_blkaddr 将被设置。
		// 如果出错或转换成功 (dn.data_blkaddr != NULL_ADDR)，则完成。
		if (err || dn.data_blkaddr != NULL_ADDR)
			goto out; // 转到清理/返回路径
		// 如果由于某种原因转换未发生但没有错误，则继续查找。
	}

	// --- 普通块查找（非内联或已转换）---
	// 首先尝试在区段缓存中查找块地址（快速路径）
	if (!f2fs_lookup_read_extent_cache_block(inode, index,
						 &dn.data_blkaddr)) {
		// --- 在区段缓存中未找到块 ---

		// 检查设备别名（特殊情况，可能是块设备支持文件？）
		if (IS_DEVICE_ALIASING(inode)) {
			err = -ENODATA; // 无法为别名设备分配？
			goto out;
		}

		// 如果我们之前获取了映射锁：
		if (locked) {
			// 尝试为此索引预留/分配一个块，因为它是一个空洞
			// 或者我们正在扩展文件。这将更新 dn.data_blkaddr（可能为 NEW_ADDR）
			// 并在元数据被修改时设置 dn.node_changed。
			err = f2fs_reserve_block(&dn, index);
			goto out; // 转到清理/返回路径
		}

		/* 空洞情况 - 最初未持有锁 */
		// 我们最初没有锁定，意味着我们不确定是否需要分配。
		// 现在我们知道它是一个空洞（非内联，不在区段缓存中）。
		// 在磁盘上的节点结构中执行完整查找。
		err = f2fs_get_dnode_of_data(&dn, index, LOOKUP_NODE);
		// 如果查找成功（无错误）并且找到了现有块 (dn.data_blkaddr != NULL_ADDR)：
		if (!err && dn.data_blkaddr != NULL_ADDR)
			goto out; // 找到了块，转到清理/返回
		// 如果查找失败或确认它是一个空洞 (dn.data_blkaddr == NULL_ADDR)：
		f2fs_put_dnode(&dn); // 释放失败/空洞查找的资源
		// 现在我们*必须*获取锁才能执行分配。
		f2fs_map_lock(sbi, F2FS_GET_BLOCK_PRE_AIO);
		WARN_ON(flag != F2FS_GET_BLOCK_PRE_AIO); // 健全性检查标志一致性
		locked = true; // 标记锁已持有
		goto restart; // 从获取 inode 页开始重新启动过程，现在持有锁。
	}
	// --- 找到块（在区段缓存中或转换/预留后）---
out:
	if (!err) { // 如果在查找/预留/转换期间未发生错误
		/* convert_inline_page 可以导致 node_changed */
		// 根据 'dn' 中的结果设置输出块地址
		*blk_addr = dn.data_blkaddr; // 可能是现有地址或 NEW_ADDR
		// 根据 'dn' 中的结果设置输出 node_changed 标志
		*node_changed = dn.node_changed;
	}
	f2fs_put_dnode(&dn); // 释放 dnode 资源（例如，节点页引用）
unlock_out:
	if (locked) // 如果我们获取了锁
		f2fs_map_unlock(sbi, flag); // 释放锁
	return err; // 返回错误码或 0 表示成功
}
```
我们先聚焦于inline数据的处理之中。
```C
// prepare_write_begin: 用于普通文件的 f2fs_write_begin 辅助函数
static int prepare_write_begin(struct f2fs_sb_info *sbi,
			struct folio *folio, loff_t pos, unsigned int len,
			block_t *blk_addr, bool *node_changed)/*最诡异的是node_changed这个变量指示是否要对文件系统做平衡操作?*/
{
	/*核心还是要将pos和len转成f2fs_mapblocks 为此我们想想当有inline数据的时候 首先是看这种情况是否有特殊的控制f2fs_mapblocks的flag。
	然后问题来了。inode中有内联数据但是不一定全不是内联数据吧。这没法保证吧。那我就得对此进行判断。我们应该能分为四种情况:*/
	/*
	1.pos+len>MAX_INODE_INLINE_DATA,也就是说写入的位置超过了最大的INLINE数据范围。注意在f2fs的上下文语境之中,这个pos指的应该是相对于inline
data 的起始的位置的偏移量 这个时候设置map block字段为default 我们看一下接下来的逻辑会怎么走。
注意它和第二个分支条件不是一样的。因为f2fs这里弄得很恶心。它有时候想以MAX_INODE_INLINE_DATA为单位判断inline数据,有时候又以i_size_read为单位判断数据。
pos+len>MAX_INODE_INLINE_DATA的情况是可以包括pos>i_size_read的 但是pos>i_size_read却很多时候不会包括前者。*/
	if (f2fs_has_inline_data(inode)) {
		if (pos + len > MAX_INLINE_DATA(inode))
			flag = F2FS_GET_BLOCK_DEFAULT;
		f2fs_map_lock(sbi, flag); // 获取锁
		locked = true;
	}
	restart: // 如果需要，在获取锁后重试的标签
	/* 检查内联数据 */
	// 获取包含 inode 直接/间接指针的节点页。
	ipage = f2fs_get_node_page(sbi, inode->i_ino);
	if (IS_ERR(ipage)) { // 获取 inode 节点页失败
		err = PTR_ERR(ipage);
		goto unlock_out;
	}
	// 使用 inode 及其节点页初始化 dnode_of_data 结构
	set_new_dnode(&dn, inode, ipage, ipage, 0); // 此处 ipage 既是 nid_page 也是 node_page
	/*memset(dn, 0, sizeof(*dn));
	dn->inode = inode;
	dn->inode_page = ipage;
	dn->node_page = npage;/*也就是node_page和ipage重合了*/
	dn->nid = nid;/*被初始化为0*/
	/*基本可以实锤一点dn里存的就是当前的页索引对应的数据块。*/
	// --- 处理内联数据 ---
	if (f2fs_has_inline_data(inode)) {
		// 写入超出了内联数据容量，需要转换为普通块
		// 此函数分配第一个数据块并将内联数据移动到其中。
		err = f2fs_convert_inline_page(&dn, folio_page(folio, 0));
		
		/*这里说一下整个转换的流程。首先为dn预留一个块然后根据dn中存的inode。f2fs_reserve_block会将dn的blk_addr设置为NEW_ADDR
		将inode的数据拷贝到传进来的folio之中(会直接调用f2fs_do_read_inline_data),然后将folio标记为脏
		然后将脏标记清理掉 进入同步回写状态。
		首先我们要理解对于buffer write为什么也要出现读的io。因为folio可能是新分配的。这意味着从内存的角度来说它不知道
		文件系统的数据是什么,如果用户写入的数据不足整个页,比如写入了开头的50字节,但是page cache中剩余的数据全是0。这个时候如果从			page cache写回到磁盘,会直接导致磁盘数据被覆盖掉。但是对于inline数据,f2fs立刻进行脏页写回的目的是什么还未知。
		*/
		// 如果转换发生，dn.data_blkaddr 将被设置。
		// 如果出错或转换成功 (dn.data_blkaddr != NULL_ADDR)，则完成。
		if (err || dn.data_blkaddr != NULL_ADDR)
			goto out; // 转到清理/返回路径
		// 如果由于某种原因转换未发生但没有错误，则继续查找。
	}
	out:
	if (!err) { // 如果在查找/预留/转换期间未发生错误
		/* convert_inline_page 可以导致 node_changed */
		// 根据 'dn' 中的结果设置输出块地址
		*blk_addr = dn.data_blkaddr; // 可能是现有地址或 NEW_ADDR
		// 根据 'dn' 中的结果设置输出 node_changed 标志
		*node_changed = dn.node_changed;
	}

}
```
