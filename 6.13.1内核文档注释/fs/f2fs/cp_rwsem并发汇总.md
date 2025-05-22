f2fs中四处遍地使用的f2fs_lock_op确实会令人疑惑它的作用。这篇md汇总一下f2fs_lock_op以及其操作的锁cp_rwsem的作用。然后我们一个一个梳理所有操作cp_rwsem的函数。注意f2fs_lock_op只是上了读锁。上写锁的函数是:
```C
static inline void f2fs_lock_all(struct f2fs_sb_info *sbi)
{
	f2fs_down_write(&sbi->cp_rwsem);
}   
```
这个锁只在block_operation中用到。会直接把所有的文件系统的操作给阻断掉。而block_operation是在f2fs_write_checkpoint函数中被调用的。
我们看f2fs_do_write_data_page中的上锁逻辑:
```C
/*按照f2fs_write_single_data_page的逻辑,普通的文件块是lock_retry*/
/*注意前面已经处理了inplace_write的情况*/
if (fio->need_lock == LOCK_RETRY) {
    /*只有当write_checkpoint函数发生了,才会出现try_lock_op失败的情况。返回的err又是-EAGAIN,在f2fs_write_single_data_page那里的话造成的效果是走分支redirty_out。也就是重新标记为脏然后解锁。*/
		if (!f2fs_trylock_op(fio->sbi)) {
			err = -EAGAIN;
			goto out_writepage;
		}
		fio->need_lock = LOCK_REQ;
	}
```
在`f2fs_do_write_data_page`中,对于普通数据的话,cp_rwsem读锁是必上的。因为后面一定会涉及到`outplace_write`。这个函数里面一定会调用`f2fs_allocate_data_block`。这个函数直接操纵的就是segment,sit等底层数据结构。所以说所有调分配块的函数都会上cp_rwsem读锁。
我们再看在`f2fs_write_compressed_pages`中的例子:
```C
static int f2fs_write_compressed_pages(struct compress_ctx *cc,
					int *submitted,
					struct writeback_control *wbc,enum iostat_type io_type)
{
	if (!f2fs_trylock_op(sbi))
	{
		goto out_free;
	}
	/*这之后执行了整个所有写压缩的全部过程。
	包括先防御性查找簇中的所有块看看是不是空,然后分配cic内存并把cc中的rpage复制给cic
	以及遍历整个cic中所有的压缩page然后全部写入磁盘的所有过程。*/
out_put_dnode: 
	f2fs_put_dnode(&dn);
out_unlock_op: 
	// 移除锁释放
	// if (quota_inode)
	// 	f2fs_up_read(&sbi->node_write);
	// else
	f2fs_unlock_op(sbi);
}
```
这个时候啊,我认为除了说`f2fs_lock_op`要保护outplace_write中的块分配逻辑以外啊,整个上锁和解锁区间中的代码实际上被视作一个原子事务了。(针对和write_checkpoint的并发来说)。毕竟压缩簇的写回磁盘一次都是以一个簇为单位的。看来f2fs不想发生压缩簇中部分页被写回部分页写回之后部分页因为cp_rwsem被锁了然后被迫放弃写回这样的中间情况。