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
if (fio->need_lock == LOCK_RETRY) {
    /*只有当write_checkpoint函数发生了,才会出现try_lock_op失败的情况。返回的err又是-EAGAIN,在f2fs_write_single_data_page那里的话造成的效果是走分支redirty_out。也就是重新标记为脏然后解锁。*/
		if (!f2fs_trylock_op(fio->sbi)) {
			err = -EAGAIN;
			goto out_writepage;
		}
		fio->need_lock = LOCK_REQ;
	}
```