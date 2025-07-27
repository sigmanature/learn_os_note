## 功能背景
保持不变
## 重大问题分析
我们已经在问题背景中分析过f2fs_iomap_folio_state的必要性 没有它导致的问题触发现象,就是我们在任何需要使用f2fs_iomap_folio_state地方的代码,以及iomap框架中需要访问iomap_folio_state地方的代码,会将其误认为是一个指针值 导致内核在形式上抛出空指针解引用的异常出来。而实际我们将我们的f2fs_iomap_folio_state集成到我们的代码的时候,我发现真是牵一发而动全身。任何原先使用旧的set_page_private/get_page_private的代码路径全部都需要更改。这里面涉及到的代码路径不光有普通数据folio 还有元数据folio中存储自己private字段的地方也需要替换成我们新设计的接口。
举个例子，page cache中的f2fs_release_folio,f2fs_is_compressed_folio. 我们顺带一下它们的函数调用栈 用dump stack来画
同时，我最终还惊讶的发现，初赛时我认为本可以和iomap框架自身保持一致地不为0阶folio分配f2fs_iomap_folio_state，然而 在我遭遇了下面这个问题之后,我的想法彻底改变了:
（问题显现)
```C
move_data_page
static int move_data_page(struct inode *inode, block_t bidx, int gc_type,
						unsigned int segno, int off)
{
	struct folio *folio;
	int err = 0;

	folio = f2fs_get_lock_data_folio(inode, bidx, true);
	if (IS_ERR(folio))
		return PTR_ERR(folio);

	if (!check_valid_map(F2FS_I_SB(inode), segno, off)) {
		err = -ENOENT;
		goto out;
	}

	err = f2fs_gc_pinned_control(inode, gc_type, segno);
	if (err)
		goto out;

	if (gc_type == BG_GC) {
		if (folio_test_writeback(folio)) {
			err = -EAGAIN;
			goto out;
		}
		folio_mark_dirty(folio);
		set_page_private_gcing(&folio->page);
	} else {
		struct f2fs_io_info fio = {
			.sbi = F2FS_I_SB(inode),
			.ino = inode->i_ino,
			.type = DATA,
			.temp = COLD,
			.op = REQ_OP_WRITE,
			.op_flags = REQ_SYNC,
			.old_blkaddr = NULL_ADDR,
			.page = &folio->page,
			.encrypted_page = NULL,
			.need_lock = LOCK_REQ,
			.io_type = FS_GC_DATA_IO,
		};
		bool is_dirty = folio_test_dirty(folio);

retry:
		f2fs_folio_wait_writeback(folio, DATA, true, true);

		folio_mark_dirty(folio);
		if (folio_clear_dirty_for_io(folio)) {
			inode_dec_dirty_pages(inode);
			f2fs_remove_dirty_inode(inode);
		}

		set_page_private_gcing(&folio->page);

		err = f2fs_do_write_data_page(&fio);
		if (err) {
			clear_page_private_gcing(&folio->page);
			if (err == -ENOMEM) {
				memalloc_retry_wait(GFP_NOFS);
				goto retry;
			}
			if (is_dirty)
				folio_mark_dirty(folio);
		}
	}
out:
	f2fs_folio_put(folio, true);
	return err;
}
```
（下面我们详细地描述一下整个gc进程先触发 设置了folio的private字段 然后我们发现在iomap_buffered_write之中我们永远不能像我们在自己使用f2fs_iomap_folio_state的时候 区分出来这里面是一个单纯的private flag字段而不是一个指针 因此是必定会引起空指针解引用崩溃的）
## 方案设计 (这里面再细分若干个小节但是目录那里可以不展示出来吧)
(这里附上整个我的f2fs_ifs.h和f2fs_ifs.c的代码 作为提供给ai的上下文)
```C
#ifndef F2FS_IFS_H
#define F2FS_IFS_H

#include <linux/fs.h>
#include <linux/bug.h>
#include <linux/f2fs_fs.h>
#include <linux/mm.h>
#include <linux/iomap.h>
#include <linux/slab.h>
#include <linux/spinlock.h>
#include <linux/atomic.h> // For atomic_t and bitops
#include "f2fs.h"
#define F2FS_IFS_MAGIC 0xf2f5 
#define F2FS_IFS_PRIVATE_LONGS 2
/*
 * F2FS structure for folio private data, mimicking iomap_folio_state layout.
 * F2FS private flags/data are stored in extra space allocated at the end
 */
struct f2fs_iomap_folio_state {
	spinlock_t state_lock;
	unsigned int read_bytes_pending;
	atomic_t write_bytes_pending; 
	/*
	 * Flexible array member.
	 * Holds [0...iomap_longs-1] for iomap uptodate/dirty bits.
	 * Holds [iomap_longs] for F2FS private flags/data (unsigned long).
	 * Holds [iomap_longs+1] for dirty_bytes_pending
	 */
	unsigned long state[];
};
static inline bool f2fs_ifs_block_is_uptodate(struct f2fs_iomap_folio_state *ifs,
		unsigned int block)
{
	return test_bit(block, ifs->state);
}
static inline size_t f2fs_ifs_iomap_longs(struct folio *folio)
{
	struct inode* inode=folio->mapping->host;
	WARN_ON_ONCE(inode==NULL);
	unsigned int nr_blocks = i_blocks_per_folio(inode, folio);
	return BITS_TO_LONGS(2 * nr_blocks);
}
static inline size_t f2fs_ifs_total_longs(struct folio *folio)
{
	return f2fs_ifs_iomap_longs(folio) + F2FS_IFS_PRIVATE_LONGS;
}

static inline unsigned long *
f2fs_ifs_private_flags_ptr(struct f2fs_iomap_folio_state *fifs, struct folio *folio)
{
	return &fifs->state[f2fs_ifs_iomap_longs(folio)];
}
static inline atomic_t *
f2fs_ifs_dirty_bytes_pending_ptr(struct f2fs_iomap_folio_state *fifs, struct folio *folio)
{
	// Treat the second private long as an atomic_t
	return (atomic_t *)&fifs->state[f2fs_ifs_iomap_longs(folio) + 1];
}

/*Have to set parameter ifs's type to void*
and have to interpret ifs as f2fs_ifs to acess it's fields because
we cannot see iomap_folio_state definition*/
static void ifs_to_f2fs_ifs(void *ifs, struct f2fs_iomap_folio_state *fifs, struct folio *folio)
{
	struct f2fs_iomap_folio_state *src_ifs = (struct f2fs_iomap_folio_state *)ifs;
	size_t iomap_longs = f2fs_ifs_iomap_longs(folio);
	fifs->read_bytes_pending = READ_ONCE(src_ifs->read_bytes_pending);
	atomic_set(&fifs->write_bytes_pending, atomic_read(&src_ifs->write_bytes_pending));
	memcpy(fifs->state, src_ifs->state, iomap_longs * sizeof(unsigned long));
}
struct f2fs_iomap_folio_state *f2fs_ifs_alloc(struct folio *folio, gfp_t gfp,bool force_alloc);
void f2fs_ifs_free(struct folio *folio);
struct f2fs_iomap_folio_state *f2fs_folio_get_private(struct folio *folio);
inline unsigned long f2fs_get_folio_private_data(struct folio *folio);
inline int f2fs_set_folio_private_data(struct folio *folio,unsigned long data);
inline void f2fs_clear_folio_private_data(struct folio *folio);
inline void f2fs_clear_folio_private_all(struct folio *folio);
/*0-order and fully dirty folio has no fifs
they store private flag directly in their folio->private field
as original f2fs page private behaviour*/
unsigned f2fs_iomap_find_dirty_range(struct folio *folio, u64 *range_start,u64 range_end);
void f2fs_ifs_clear_range_uptodate(struct folio *folio, struct f2fs_iomap_folio_state*fifs,size_t off, size_t len);
void f2fs_iomap_finish_folio_read(struct folio *folio, size_t off,size_t len, int error);
static inline bool is_f2fs_ifs(struct folio *folio)
{
    if (!folio_test_private(folio))
        return false;
        
    // first directly test no pointer flag is set or not
    if (test_bit(PAGE_PRIVATE_NOT_POINTER, (unsigned long *)&folio->private))
        return false;
        
    struct f2fs_iomap_folio_state *fifs;
    fifs = (struct f2fs_iomap_folio_state *)folio->private;
    if (!fifs)
        return false;
    if (READ_ONCE(fifs->read_bytes_pending) == F2FS_IFS_MAGIC) {
        return true;
    }
    return false;
}
#define F2FS_FOLIO_PRIVATE_GET_FUNC(name, flagname)                                \
	static inline bool f2fs_folio_private_##name(struct folio *folio)          \
	{   																	\
		/* First try direct folio->private access for meta folio */                       \
		if (folio_test_private(folio) &&                                   \
		    test_bit(PAGE_PRIVATE_NOT_POINTER,                             \
			     (unsigned long *)&folio->private)) {                  \
			return test_bit(PAGE_PRIVATE_##flagname,                   \
					(unsigned long *)&folio->private);        \
		}																	\
		/* For higher-order folios, use iomap folio state */               \
		struct f2fs_iomap_folio_state *fifs =                              \
			(struct f2fs_iomap_folio_state *)folio->private;           \
		unsigned long *private_p;                                          \
		if (unlikely(!fifs || !folio->mapping))                            \
			return false;                                               \
		/* Check magic number before accessing private data */             \
		if (READ_ONCE(fifs->read_bytes_pending) != F2FS_IFS_MAGIC)         \
			return false;                                               \
		private_p = f2fs_ifs_private_flags_ptr(fifs, folio);               \
		if (!private_p)                                                    \
			return false;                                               \
		/* Test bits directly on the 'private' slot */                     \
		return test_bit(PAGE_PRIVATE_##flagname, private_p);               \
	}

#define F2FS_FOLIO_PRIVATE_SET_FUNC(name, flagname)                               \
	static inline int f2fs_set_folio_private_##name(struct folio *folio)      \
	{                                                                         \
		/* For higher-order folios, use iomap folio state */             \
		if (unlikely(!folio->mapping))                                   \
			return -ENOENT;													\
		bool force_alloc=f2fs_should_use_buffered_iomap(folio_inode(folio)); \
		if (!force_alloc && !folio_test_private(folio)) {                 \
			folio_attach_private(folio, (void *)0);                   \
			set_bit(PAGE_PRIVATE_NOT_POINTER,                         \
				(unsigned long *)&folio->private);               \
			set_bit(PAGE_PRIVATE_##flagname,                          \
				(unsigned long *)&folio->private);               \
			return 0;                                                 \
		}     															\
		struct f2fs_iomap_folio_state *fifs =                            \
			f2fs_ifs_alloc(folio, GFP_NOFS,true);                         \
		if(unlikely(!fifs))                                              \
			return -ENOMEM;                                           \
		unsigned long *private_p;                                        \
		WRITE_ONCE(fifs->read_bytes_pending, F2FS_IFS_MAGIC);            \
		private_p = f2fs_ifs_private_flags_ptr(fifs, folio);             \
		if (!private_p)                                                  \
			return -EINVAL;                                           \
		/* Set the bit atomically */                                     \
		set_bit(PAGE_PRIVATE_##flagname, private_p);                     \
		/* Ensure NOT_POINTER bit is also set if any F2FS flag is set */ \
		if (PAGE_PRIVATE_##flagname != PAGE_PRIVATE_NOT_POINTER)         \
			set_bit(PAGE_PRIVATE_NOT_POINTER, private_p);            \
		return 0;                                                        \
	}

#define F2FS_FOLIO_PRIVATE_CLEAR_FUNC(name, flagname)                         \
	static inline void f2fs_clear_folio_private_##name(                   \
		struct folio *folio)                                          \
	{                            										\
			/* First try direct folio->private access */                  \
		if (folio_test_private(folio) &&                              \
		    test_bit(PAGE_PRIVATE_NOT_POINTER,                        \
			     (unsigned long *)&folio->private)) {             \
			clear_bit(PAGE_PRIVATE_##flagname,                    \
				  (unsigned long *)&folio->private);          \
			folio_detach_private(folio);                  \
			return;                                               \
		}                                                             \
		/* For higher-order folios, use iomap folio state */         \
		struct f2fs_iomap_folio_state *fifs =                         \
			(struct f2fs_iomap_folio_state *)folio->private;      \
		unsigned long *private_p;                                     \
		if (unlikely(!fifs || !folio->mapping))                       \
			return;                                               \
		/* Check magic number before clearing */                      \
		if (READ_ONCE(fifs->read_bytes_pending) != F2FS_IFS_MAGIC)    \
			return; /* Not ours or state unclear */               \
		private_p = f2fs_ifs_private_flags_ptr(fifs, folio);          \
		if (!private_p)                                               \
			return;                                               \
		clear_bit(PAGE_PRIVATE_##flagname, private_p);                \
	}

// Generate the accessor functions using the macros
F2FS_FOLIO_PRIVATE_GET_FUNC(nonpointer, NOT_POINTER);
F2FS_FOLIO_PRIVATE_GET_FUNC(inline, INLINE_INODE);
F2FS_FOLIO_PRIVATE_GET_FUNC(gcing, ONGOING_MIGRATION);
F2FS_FOLIO_PRIVATE_GET_FUNC(atomic, ATOMIC_WRITE);
F2FS_FOLIO_PRIVATE_GET_FUNC(reference, REF_RESOURCE);
F2FS_FOLIO_PRIVATE_GET_FUNC(deferred_unlock, DEFERRED_UNLOCK);

F2FS_FOLIO_PRIVATE_SET_FUNC(reference, REF_RESOURCE);
F2FS_FOLIO_PRIVATE_SET_FUNC(inline, INLINE_INODE);
F2FS_FOLIO_PRIVATE_SET_FUNC(gcing, ONGOING_MIGRATION);
F2FS_FOLIO_PRIVATE_SET_FUNC(atomic, ATOMIC_WRITE);
F2FS_FOLIO_PRIVATE_SET_FUNC(deferred_unlock, DEFERRED_UNLOCK);

F2FS_FOLIO_PRIVATE_CLEAR_FUNC(reference, REF_RESOURCE);
F2FS_FOLIO_PRIVATE_CLEAR_FUNC(inline, INLINE_INODE);
F2FS_FOLIO_PRIVATE_CLEAR_FUNC(gcing, ONGOING_MIGRATION);
F2FS_FOLIO_PRIVATE_CLEAR_FUNC(atomic, ATOMIC_WRITE);
F2FS_FOLIO_PRIVATE_CLEAR_FUNC(deferred_unlock, DEFERRED_UNLOCK);
// --- Data access functions ---
#endif /* F2FS_IFS_H */

//f2fs_ifs.c
#include <linux/fs.h>
#include <linux/f2fs_fs.h>
#include <linux/mm.h>
#include <linux/iomap.h>
#include <linux/slab.h>
#include <linux/sched.h> // For cond_resched()
#include "f2fs.h"
#include "f2fs_ifs.h" 
struct f2fs_iomap_folio_state *f2fs_ifs_alloc(struct folio *folio, gfp_t gfp,bool force_alloc)
{
	struct inode* inode= folio->mapping->host;
	size_t alloc_size=0;
	if (folio_order(folio) == 0) {
		if(!force_alloc)
		{
			WARN_ON_ONCE(1); 
			return NULL;
		}
		else
		{/* GC can store private flag in 0 order folio's folio->private
			causes iomap buffered write mistakenly interpret as a pointer
			we add a bool force_alloc to deal with this case
		*/
			struct f2fs_iomap_folio_state *fifs;
			alloc_size = sizeof(*fifs) + 2*sizeof(unsigned long);
			fifs = kmalloc(alloc_size, gfp); 
			if (!fifs)
				return NULL;
			spin_lock_init(&fifs->state_lock);
			WRITE_ONCE(fifs->read_bytes_pending, F2FS_IFS_MAGIC);
        	atomic_set(&fifs->write_bytes_pending, 0); 
			unsigned int nr_blocks = i_blocks_per_folio(inode, folio);
			if (folio_test_uptodate(folio))
				bitmap_set(fifs->state, 0, nr_blocks);
			if (folio_test_dirty(folio))
				(fifs->state, nr_blocks, nr_blocks);
			*f2fs_ifs_private_flags_ptr(fifs, folio) = 0; 
			folio_attach_private(folio, fifs);
		}
	}
	struct f2fs_iomap_folio_state *fifs;
	void *old_private;
	size_t iomap_longs;
	size_t total_longs;	
	WARN_ON_ONCE(!inode); // Should have an inode

	old_private = folio_get_private(folio);

	if (old_private) {
		// Check if it's already our type using the magic number directly
		if (READ_ONCE(((struct f2fs_iomap_folio_state *)old_private)->read_bytes_pending) == F2FS_IFS_MAGIC) {
			return (struct f2fs_iomap_folio_state *)old_private; // Already ours
		} else {
			// Non-NULL, not ours -> Allocate, Copy, Replace path
			total_longs = f2fs_ifs_total_longs(folio);
			alloc_size = sizeof(*fifs) + total_longs * sizeof(unsigned long);

			fifs = kmalloc(alloc_size, gfp); 
			if (!fifs)
				return NULL;

			spin_lock_init(&fifs->state_lock);
			*f2fs_ifs_private_flags_ptr(fifs, folio) = 0; 
			// Copy data from the presumed iomap_folio_state (old_private)
			ifs_to_f2fs_ifs(old_private, fifs, folio);
			WRITE_ONCE(fifs->read_bytes_pending, F2FS_IFS_MAGIC);
            atomic_set(f2fs_ifs_dirty_bytes_pending_ptr(fifs, folio), 0);
			folio_change_private(folio, fifs);
			kfree(old_private); 
			return fifs;
		}
	} else {
		iomap_longs = f2fs_ifs_iomap_longs(folio);
		total_longs = iomap_longs + 1;
		alloc_size = sizeof(*fifs) + total_longs * sizeof(unsigned long);

		fifs = kzalloc(alloc_size, gfp);
		if (!fifs)
			return NULL;

		spin_lock_init(&fifs->state_lock);

		unsigned int nr_blocks = i_blocks_per_folio(inode, folio);
		
		if (folio_test_uptodate(folio))
			bitmap_set(fifs->state, 0, nr_blocks);
		if (folio_test_dirty(folio))
			bitmap_set(fifs->state, nr_blocks, nr_blocks);		
		WRITE_ONCE(fifs->read_bytes_pending, F2FS_IFS_MAGIC);
        atomic_set(&fifs->write_bytes_pending, 0); 
        atomic_set(f2fs_ifs_dirty_bytes_pending_ptr(fifs, folio), 0);
		folio_attach_private(folio, fifs);
		return fifs;
	}
}
void f2fs_ifs_free(struct folio *folio)
{
	struct f2fs_iomap_folio_state*fifs;
	
	if (!folio_test_private(folio))
		return;
		
	// Check if it's using direct flags
	if (test_bit(PAGE_PRIVATE_NOT_POINTER, (unsigned long *)&folio->private)) {
		folio_detach_private(folio);
		return;
	}
	
	fifs = folio_detach_private(folio);
	if (!fifs)
		return; 
	
	#ifdef CONFIG_F2FS_DEBUG_PRINT
		f2fs_err(F2FS_I_SB(folio->mapping->host),"f2fs_ifs_free: %d, read_bytes_pending %x, write_bytes_pending %x",
			folio_order(folio), READ_ONCE(fifs->read_bytes_pending),
			atomic_read(&fifs->write_bytes_pending));
	#endif
	
	if(is_f2fs_ifs(folio))
	{	
		WARN_ON_ONCE(READ_ONCE(fifs->read_bytes_pending) != F2FS_IFS_MAGIC);
		WARN_ON_ONCE(atomic_read(&fifs->write_bytes_pending));
	}
	else
	{
		WARN_ON_ONCE(READ_ONCE(fifs->read_bytes_pending) != 0);
		WARN_ON_ONCE(atomic_read(&fifs->write_bytes_pending));
	}
	
	kfree(fifs);
}
struct f2fs_iomap_folio_state *f2fs_folio_get_private(struct folio *folio)
{
    if (!folio_test_private(folio))
        return NULL;
        
    // 检查是否为标志位使用
    if (test_bit(PAGE_PRIVATE_NOT_POINTER, (unsigned long *)&folio->private))
        return NULL;
        
    void *private_data = folio->private;
    if (!private_data)
        return NULL;
        
    // 安全地检查 magic
    struct f2fs_iomap_folio_state *fifs = private_data;
    if (READ_ONCE(fifs->read_bytes_pending) == F2FS_IFS_MAGIC)
        return fifs;
    
    return NULL;
}
__attribute__((optimize("O0")))
unsigned f2fs_iomap_find_dirty_range(struct folio *folio, u64 *range_start,
		u64 range_end)
{
	struct inode* inode=folio->mapping->host;
	
	if(folio_order(folio) == 0) 
	 	return range_end-*range_start;
	if(f2fs_compressed_file(inode))
	{	
		/*clamp range end to a cluster's size*/
		int a=*range_start>>PAGE_SHIFT;
		int b=cluster_i_idx(inode, a)<<F2FS_I(inode)->i_cluster_size;
		int c=(F2FS_I(inode)->i_cluster_size - 1);
		range_end=min(range_end,(i_end_idx_of_cluster(inode,*range_start>>PAGE_SHIFT)+1) << PAGE_SHIFT);
	}
	return iomap_find_dirty_range(folio, range_start, range_end);
}
void f2fs_ifs_clear_range_uptodate(struct folio *folio, struct f2fs_iomap_folio_state*fifs,size_t off, size_t len)
{
	struct inode *inode = folio->mapping->host;
	unsigned int first_blk = (off >> inode->i_blkbits);
	unsigned int last_blk = (off + len - 1) >> inode->i_blkbits;
	unsigned int nr_blks = last_blk - first_blk + 1;
	unsigned long flags;
	spin_lock_irqsave(&fifs->state_lock, flags);
	bitmap_clear(fifs->state, first_blk, nr_blks);
	spin_unlock_irqrestore(&fifs->state_lock, flags);
}
inline unsigned long f2fs_get_folio_private_data(struct folio *folio)
{
	struct f2fs_iomap_folio_state *fifs = 
		(struct f2fs_iomap_folio_state *)folio->private;
	unsigned long *private_p;
	unsigned long data_val;
	if (!folio->mapping)
		return 0;
	f2fs_bug_on(F2FS_I_SB(folio_inode(folio)),!fifs);
	if (READ_ONCE(fifs->read_bytes_pending) != F2FS_IFS_MAGIC)
		return 0;

	private_p = f2fs_ifs_private_flags_ptr(fifs, folio);
	if (!private_p)
		return 0;

	data_val = READ_ONCE(*private_p); // Read atomically

	if (!test_bit(PAGE_PRIVATE_NOT_POINTER, &data_val))
		return 0; // Return 0 if NOT_POINTER isn't set

	return data_val >> PAGE_PRIVATE_MAX;
}

inline int f2fs_set_folio_private_data(struct folio *folio, unsigned long data)
{
	
	if (unlikely(!folio->mapping))
		return -ENOENT;
	
	struct f2fs_iomap_folio_state *fifs = f2fs_ifs_alloc(folio, GFP_NOFS, true);
	if (unlikely(!fifs))
		return -ENOMEM;
	
	unsigned long *private_p;
	unsigned long old_val, new_val;
	
	private_p = f2fs_ifs_private_flags_ptr(fifs, folio);
	if (!private_p)
		return -EINVAL;

	// Atomically set the data part and the NOT_POINTER bit using cmpxchg loop
	do {
		old_val = READ_ONCE(*private_p);
		new_val = old_val;
		// Clear old data bits (bits >= PAGE_PRIVATE_MAX)
		new_val &= GENMASK(PAGE_PRIVATE_MAX - 1, 0);
		// Set new data bits
		new_val |= (data << PAGE_PRIVATE_MAX);
		// Ensure NOT_POINTER is set
		__set_bit(PAGE_PRIVATE_NOT_POINTER, &new_val);
	} while (cmpxchg(private_p, old_val, new_val) != old_val);

	return 0;
}

inline void f2fs_clear_folio_private_data(struct folio *folio)
{	
	struct f2fs_iomap_folio_state *fifs = 
		(struct f2fs_iomap_folio_state *)folio->private;
	unsigned long *private_p;
	unsigned long old_val, new_val;
	f2fs_bug_on(F2FS_I_SB(folio_inode(folio)),!fifs);
	if (!folio->mapping)
		return;
	if (READ_ONCE(fifs->read_bytes_pending) != F2FS_IFS_MAGIC)
		return;

	private_p = f2fs_ifs_private_flags_ptr(fifs, folio);
	if (!private_p)
		return;

	// Atomically clear the data part, leave flags untouched
	do {
		old_val = READ_ONCE(*private_p);
		// If already no data bits set, nothing to do
		if ((old_val >> PAGE_PRIVATE_MAX) == 0)
			break;
		new_val = old_val;
		// Clear data bits
		new_val &= GENMASK(PAGE_PRIVATE_MAX - 1, 0);
	} while (cmpxchg(private_p, old_val, new_val) != old_val);
}

inline void f2fs_clear_folio_private_all(struct folio *folio)
{
	struct f2fs_iomap_folio_state *fifs = 
		(struct f2fs_iomap_folio_state *)folio->private;
	unsigned long *private_p;

	if (unlikely(!fifs || !folio->mapping))
		return;
	if (READ_ONCE(fifs->read_bytes_pending) != F2FS_IFS_MAGIC)
		return;

	private_p = f2fs_ifs_private_flags_ptr(fifs, folio);
	if (!private_p)
		return;
		
	// Clear all private flags/data
	*private_p = 0;
}
```
首先f2fs_ifs结构体的各种定义是保持不变的。但是我我们描述f2fsfolio private api的时候
要进行一个较大的改动了。大体的讲法:取我们的任何一个f2fs_get/set_folio_private作为示例讲法。
讲解到我们不直接依赖于folio的阶数判断 而是先直接test标志位 因为这里面巧就巧在 f2fs原先的标志位设计本来就是能区分是指针还是不是指针
毕竟已经有了no_pointer了 而f2fs_iomap_folio_state又恰好是个指针
然后我们展现我们的f2fs_iomap_folio_state在各个代码路径里的具体使用,把他们给展示出来。
另外我们大致展开描述一下f2fs对page private flag的几个动作项。就是设置,获取，清除 我们稍微展开描述一下。
