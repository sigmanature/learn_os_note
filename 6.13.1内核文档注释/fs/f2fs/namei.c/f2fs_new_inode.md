 **相关函数**
 * [f2fs_may_inline_data](https://github.com/sigmanature/learn_os_note/tree/main/6.13.1%E5%86%85%E6%A0%B8%E6%96%87%E6%A1%A3%E6%B3%A8%E9%87%8A/fs/f2fs/inline.c/f2fs_may_inline_data) <br>
 Absolutely！`f2fs_new_inode` 是 F2FS 文件系统中创建新 inode 的核心函数。它负责分配 inode 编号、初始化 inode 结构、设置各种标志（包括内联数据和扩展属性相关的标志），并处理一些初始化操作。让我们逐行深入解析这个函数，重点关注你关心的内联数据、xattr 和大小设置。

```c
static struct inode *f2fs_new_inode(struct mnt_idmap *idmap,
						struct inode *dir, umode_t mode,
						const char *name)
{
	struct f2fs_sb_info *sbi = F2FS_I_SB(dir);
	struct f2fs_inode_info *fi;
	nid_t ino;
	struct inode *inode;
	bool nid_free = false;
	bool encrypt = false;
	int xattr_size = 0;
	int err;
```

* **`static struct inode *f2fs_new_inode(...)`**:  定义了 `f2fs_new_inode` 函数，它返回一个 `struct inode` 类型的指针。`static` 关键字意味着这个函数的作用域限制在当前源文件内。
* **`struct mnt_idmap *idmap, struct inode *dir, umode_t mode, const char *name`**:  函数接受四个参数：
    * `idmap`:  用于命名空间 ID 映射，在容器环境中可能用到。
    * `dir`:  指向父目录的 inode 结构。新 inode 将会在这个目录下创建。
    * `mode`:  新 inode 的模式，包括文件类型（普通文件、目录等）和权限。`umode_t` 是 unsigned mode_t 类型。
    * `name`:  新 inode 的名称（文件名或目录名）。
* **`struct f2fs_sb_info *sbi = F2FS_I_SB(dir);`**: 获取 F2FS 文件系统的超级块信息。
    * `F2FS_I_SB(dir)` 是一个宏，它从父目录 `dir` 的 inode 结构中获取 `f2fs_sb_info` 结构体的指针。`f2fs_sb_info` 包含了文件系统的全局信息，例如超级块、选项设置等。
* **`struct f2fs_inode_info *fi;`**:  声明一个指向 `f2fs_inode_info` 结构体的指针 `fi`。`f2fs_inode_info` 是 F2FS 特有的 inode 信息结构体，它嵌入在通用的 `inode` 结构体中，用于存储 F2FS 特有的 inode 元数据。
* **`nid_t ino;`**: 声明一个 `nid_t` 类型的变量 `ino`，用于存储新分配的 inode 编号 (Node ID)。`nid_t` 是 F2FS 中表示 inode 编号的类型。
* **`struct inode *inode;`**: 声明一个指向通用 `inode` 结构体的指针 `inode`。这是 Linux 内核中表示 inode 的标准结构体。
* **`bool nid_free = false;`**:  声明一个布尔变量 `nid_free` 并初始化为 `false`。这个变量用于标记是否成功分配了 inode 编号。如果分配失败，需要释放 inode 编号。
* **`bool encrypt = false;`**: 声明一个布尔变量 `encrypt` 并初始化为 `false`。用于标记是否需要对 inode 进行加密。
* **`int xattr_size = 0;`**: 声明一个整型变量 `xattr_size` 并初始化为 `0`。这个变量用于存储内联扩展属性 (inline xattr) 的大小。
* **`int err;`**: 声明一个整型变量 `err`，用于存储错误码。

```c
	inode = new_inode(dir->i_sb);
	if (!inode)
		return ERR_PTR(-ENOMEM);
```

* **`inode = new_inode(dir->i_sb);`**:  分配一个新的通用 `inode` 结构体。
    * `new_inode(dir->i_sb)` 是 Linux 内核提供的函数，用于分配一个新的 `inode` 结构体。它会从 inode 缓存中分配，并初始化一些基本字段。`dir->i_sb` 是父目录 inode 的超级块，用于指定 inode 所属的文件系统。
* **`if (!inode) return ERR_PTR(-ENOMEM);`**: 错误处理。如果 `new_inode` 返回 `NULL` (分配失败)，则返回 `-ENOMEM` 错误，表示内存不足。`ERR_PTR` 是一个宏，用于将错误码转换为错误指针。

```c
	if (!f2fs_alloc_nid(sbi, &ino)) {
		err = -ENOSPC;
		goto fail;
	}

	nid_free = true;
```

* **`if (!f2fs_alloc_nid(sbi, &ino))`**:  分配一个新的 inode 编号 (NID)。
    * `f2fs_alloc_nid(sbi, &ino)` 是 F2FS 特有的函数，用于从文件系统的 inode 位图中分配一个新的 inode 编号。`sbi` 是文件系统的超级块信息，`&ino` 是用于接收分配到的 inode 编号的指针。
    * 如果分配失败（例如，inode 位图已满），函数返回 `false`。
* **`err = -ENOSPC; goto fail;`**:  如果 inode 编号分配失败，设置错误码为 `-ENOSPC` (No space left on device)，并跳转到 `fail` 标签进行错误处理。
* **`nid_free = true;`**:  如果 inode 编号分配成功，将 `nid_free` 设置为 `true`，表示后续如果发生错误，需要释放已分配的 inode 编号。

```c
	inode_init_owner(idmap, inode, dir, mode);
```

* **`inode_init_owner(idmap, inode, dir, mode);`**: 初始化 inode 的所有者 (owner)、组 (group) 和权限。
    * `inode_init_owner` 是 Linux 内核提供的函数，用于根据父目录 `dir` 和给定的 `mode` 初始化新 inode 的 `i_uid`、`i_gid` 和权限位。`idmap` 用于处理命名空间 ID 映射。

```c
	fi = F2FS_I(inode);
	inode->i_ino = ino;
	inode->i_blocks = 0;
	simple_inode_init_ts(inode);
	fi->i_crtime = inode_get_mtime(inode);
	inode->i_generation = get_random_u32();
```

* **`fi = F2FS_I(inode);`**: 获取 F2FS 特有的 inode 信息结构体 `f2fs_inode_info` 的指针。
    * `F2FS_I(inode)` 是一个宏，它从通用的 `inode` 结构体 `inode` 中获取嵌入的 `f2fs_inode_info` 结构体的指针。
* **`inode->i_ino = ino;`**:  设置 inode 的编号。将之前分配的 inode 编号 `ino` 赋值给 `inode->i_ino`。
* **`inode->i_blocks = 0;`**: 初始化 inode 占用的块数为 0。新创建的 inode 初始状态下不占用任何数据块。
* **`simple_inode_init_ts(inode);`**: 初始化 inode 的时间戳 (timestamp)。
    * `simple_inode_init_ts` 是 Linux 内核提供的函数，用于初始化 inode 的 `i_atime` (访问时间)、`i_mtime` (修改时间) 和 `i_ctime` (状态改变时间) 为当前时间。
* **`fi->i_crtime = inode_get_mtime(inode);`**: 初始化 F2FS inode 信息结构体中的创建时间 `i_crtime`。
    * `inode_get_mtime(inode)` 获取 inode 的修改时间 (此时已经被 `simple_inode_init_ts` 初始化为当前时间)。F2FS 使用 `i_crtime` 记录 inode 的创建时间。
* **`inode->i_generation = get_random_u32();`**: 初始化 inode 的 generation number。
    * `inode->i_generation` 用于 NFS 文件系统，作为文件句柄的一部分，用于区分文件的新旧版本。`get_random_u32()` 生成一个随机的 32 位无符号整数。

```c
	if (S_ISDIR(inode->i_mode))
		fi->i_current_depth = 1;
```

* **`if (S_ISDIR(inode->i_mode))`**: 检查新创建的 inode 是否是目录。
    * `S_ISDIR(inode->i_mode)` 是一个宏，用于判断 inode 的模式 `i_mode` 是否表示目录类型。
* **`fi->i_current_depth = 1;`**: 如果是目录，则设置 F2FS inode 信息结构体中的 `i_current_depth` 为 1。
    * `i_current_depth` 用于记录目录的深度，根目录深度为 0，其子目录深度为 1，以此类推。新创建的目录深度为 1。

```c
	err = insert_inode_locked(inode);
	if (err) {
		err = -EINVAL;
		goto fail;
	}
```

* **`err = insert_inode_locked(inode);`**: 将新创建的 inode 插入到 inode 哈希表和 LRU 链表中。
    * `insert_inode_locked(inode)` 是 Linux 内核提供的函数，用于将新创建的 inode 添加到全局的 inode 管理结构中，使其可以被查找和访问。`locked` 表示这个操作需要持有 inode 相关的锁。
* **`if (err) { err = -EINVAL; goto fail; }`**: 错误处理。如果 `insert_inode_locked` 返回错误，设置错误码为 `-EINVAL` (Invalid argument)，并跳转到 `fail` 标签进行错误处理。

```c
	if (f2fs_sb_has_project_quota(sbi) &&
		(F2FS_I(dir)->i_flags & F2FS_PROJINHERIT_FL))
		fi->i_projid = F2FS_I(dir)->i_projid;
	else
		fi->i_projid = make_kprojid(&init_user_ns,
							F2FS_DEF_PROJID);
```

* **`if (f2fs_sb_has_project_quota(sbi) && (F2FS_I(dir)->i_flags & F2FS_PROJINHERIT_FL))`**: 检查是否启用项目配额 (project quota) 并且父目录是否设置了项目配额继承标志 `F2FS_PROJINHERIT_FL`。
    * `f2fs_sb_has_project_quota(sbi)` 检查文件系统是否启用了项目配额功能。
    * `F2FS_I(dir)->i_flags & F2FS_PROJINHERIT_FL` 检查父目录 inode 的 F2FS 特有标志 `i_flags` 中是否设置了 `F2FS_PROJINHERIT_FL` 标志，表示子目录或文件应该继承父目录的项目 ID。
* **`fi->i_projid = F2FS_I(dir)->i_projid;`**: 如果条件成立，则新 inode 继承父目录的项目 ID。将父目录 inode 的项目 ID `F2FS_I(dir)->i_projid` 赋值给新 inode 的项目 ID `fi->i_projid`。
* **`else fi->i_projid = make_kprojid(&init_user_ns, F2FS_DEF_PROJID);`**: 否则，使用默认的项目 ID。
    * `make_kprojid(&init_user_ns, F2FS_DEF_PROJID)` 创建一个内核项目 ID 结构体，使用初始用户命名空间 `init_user_ns` 和默认项目 ID `F2FS_DEF_PROJID`。`F2FS_DEF_PROJID` 通常是 0。

```c
	err = fscrypt_prepare_new_inode(dir, inode, &encrypt);
	if (err)
		goto fail_drop;
```

* **`err = fscrypt_prepare_new_inode(dir, inode, &encrypt);`**:  准备 inode 加密。
    * `fscrypt_prepare_new_inode(dir, inode, &encrypt)` 是文件系统加密 (fscrypt) 子系统的函数，用于检查是否需要对新 inode 进行加密，并进行必要的准备工作。`dir` 是父目录 inode，`inode` 是新 inode，`&encrypt` 是一个输出参数，用于指示是否需要加密。
* **`if (err) goto fail_drop;`**: 错误处理。如果 `fscrypt_prepare_new_inode` 返回错误，跳转到 `fail_drop` 标签进行错误处理。`fail_drop` 错误处理路径会释放配额等资源。

```c
	err = f2fs_dquot_initialize(inode);
	if (err)
		goto fail_drop;
```

* **`err = f2fs_dquot_initialize(inode);`**: 初始化 inode 的磁盘配额 (dquot)。
    * `f2fs_dquot_initialize(inode)` 是 F2FS 特有的函数，用于初始化新 inode 的磁盘配额信息。
* **`if (err) goto fail_drop;`**: 错误处理。如果 `f2fs_dquot_initialize` 返回错误，跳转到 `fail_drop` 标签进行错误处理。

```c
	set_inode_flag(inode, FI_NEW_INODE);
```

* **`set_inode_flag(inode, FI_NEW_INODE);`**: 设置 inode 的 `FI_NEW_INODE` 标志。
    * `set_inode_flag(inode, FI_NEW_INODE)` 是一个宏，用于设置 inode 的 `i_flags` 字段中的 `FI_NEW_INODE` 标志。`FI_NEW_INODE` 标志表示这是一个新创建的 inode。

```c
	if (encrypt)
		f2fs_set_encrypted_inode(inode);
```

* **`if (encrypt) f2fs_set_encrypted_inode(inode);`**: 如果 `encrypt` 为真 (表示需要加密)，则设置 inode 的加密标志。
    * `f2fs_set_encrypted_inode(inode)` 是 F2FS 特有的函数，用于设置 inode 的加密相关标志，例如 `I_ENCRYPTED` 标志。

```c
	if (f2fs_sb_has_extra_attr(sbi)) {
		set_inode_flag(inode, FI_EXTRA_ATTR);
		fi->i_extra_isize = F2FS_TOTAL_EXTRA_ATTR_SIZE;
	}
```

* **`if (f2fs_sb_has_extra_attr(sbi))`**: 检查文件系统是否支持扩展属性 (extra attribute) 功能。
    * `f2fs_sb_has_extra_attr(sbi)` 检查文件系统的超级块信息 `sbi` 中是否设置了支持扩展属性的标志。这个功能通常用于存储更大的 xattr 数据。
* **`set_inode_flag(inode, FI_EXTRA_ATTR);`**: 如果支持扩展属性，则设置 inode 的 `FI_EXTRA_ATTR` 标志。
    * `set_inode_flag(inode, FI_EXTRA_ATTR)` 设置 inode 的 `i_flags` 字段中的 `FI_EXTRA_ATTR` 标志，表示这个 inode 可以使用扩展属性功能。
* **`fi->i_extra_isize = F2FS_TOTAL_EXTRA_ATTR_SIZE;`**: 设置 F2FS inode 信息结构体中的 `i_extra_isize` 为 `F2FS_TOTAL_EXTRA_ATTR_SIZE`。
    * `i_extra_isize` 字段用于记录扩展属性区域的大小。`F2FS_TOTAL_EXTRA_ATTR_SIZE` 是一个宏，定义了扩展属性区域的总大小。**这里设置了扩展属性区域的固定大小。** 你可能需要查找 `F2FS_TOTAL_EXTRA_ATTR_SIZE` 的定义来确定具体大小。

```c
	if (test_opt(sbi, INLINE_XATTR))
		set_inode_flag(inode, FI_INLINE_XATTR);
```

* **`if (test_opt(sbi, INLINE_XATTR))`**: 检查文件系统是否启用了内联扩展属性 (inline xattr) 功能。
    * `test_opt(sbi, INLINE_XATTR)` 检查文件系统的选项中是否启用了 `INLINE_XATTR` 选项。这个选项允许将小的 xattr 数据直接存储在 inode 结构中，而不是单独分配块。
* **`set_inode_flag(inode, FI_INLINE_XATTR);`**: 如果启用了内联 xattr，则设置 inode 的 `FI_INLINE_XATTR` 标志。
    * `set_inode_flag(inode, FI_INLINE_XATTR)` 设置 inode 的 `i_flags` 字段中的 `FI_INLINE_XATTR` 标志，表示这个 inode 可以使用内联 xattr 功能。

```c
	if (f2fs_may_inline_dentry(inode))
		set_inode_flag(inode, FI_INLINE_DENTRY);
```

* **`if (f2fs_may_inline_dentry(inode))`**: 检查 inode 是否可以内联目录项 (inline dentry)。
    * `f2fs_may_inline_dentry(inode)` 是 F2FS 特有的函数，用于判断是否可以将目录项 (dentry) 内联存储在目录 inode 中。这通常用于优化小目录的性能。判断条件可能包括目录项的大小、目录 inode 的可用空间等。
* **`set_inode_flag(inode, FI_INLINE_DENTRY);`**: 如果可以内联目录项，则设置 inode 的 `FI_INLINE_DENTRY` 标志。
    * `set_inode_flag(inode, FI_INLINE_DENTRY)` 设置 inode 的 `i_flags` 字段中的 `FI_INLINE_DENTRY` 标志，表示这个 inode 可以使用内联目录项功能。

```c
	if (f2fs_sb_has_flexible_inline_xattr(sbi)) {
		f2fs_bug_on(sbi, !f2fs_has_extra_attr(inode));
		if (f2fs_has_inline_xattr(inode))
			xattr_size = F2FS_OPTION(sbi).inline_xattr_size;
		/* Otherwise, will be 0 */
	} else if (f2fs_has_inline_xattr(inode) ||
				f2fs_has_inline_dentry(inode)) {
		xattr_size = DEFAULT_INLINE_XATTR_ADDRS;
	}
	fi->i_inline_xattr_size = xattr_size;
```

* **`if (f2fs_sb_has_flexible_inline_xattr(sbi))`**: 检查文件系统是否支持灵活的内联扩展属性 (flexible inline xattr) 功能。
    * `f2fs_sb_has_flexible_inline_xattr(sbi)` 检查文件系统的超级块信息 `sbi` 中是否设置了支持灵活内联 xattr 的标志。灵活内联 xattr 允许根据文件系统选项配置内联 xattr 的大小。
* **`f2fs_bug_on(sbi, !f2fs_has_extra_attr(inode));`**:  断言检查。如果启用了灵活内联 xattr，则必须同时启用扩展属性功能。
    * `f2fs_bug_on(sbi, !f2fs_has_extra_attr(inode))` 是 F2FS 特有的断言宏，用于在条件为真时触发内核 BUG。这里检查如果启用了灵活内联 xattr，但 inode 没有启用扩展属性功能 (`!f2fs_has_extra_attr(inode)` 为真)，则触发 BUG。
* **`if (f2fs_has_inline_xattr(inode)) xattr_size = F2FS_OPTION(sbi).inline_xattr_size;`**: 如果 inode 启用了内联 xattr，并且文件系统支持灵活内联 xattr，则从文件系统选项中获取内联 xattr 的大小。
    * `f2fs_has_inline_xattr(inode)` 检查 inode 是否设置了 `FI_INLINE_XATTR` 标志。
    * `F2FS_OPTION(sbi).inline_xattr_size` 从文件系统的选项结构体 `F2FS_OPTION(sbi)` 中获取 `inline_xattr_size` 字段的值。**这里内联 xattr 的大小是可配置的，由文件系统选项 `inline_xattr_size` 决定。** 你需要查看文件系统挂载选项或者默认配置来确定这个大小。
* **`else if (f2fs_has_inline_xattr(inode) || f2fs_has_inline_dentry(inode))`**: 如果文件系统不支持灵活内联 xattr，但 inode 启用了内联 xattr 或内联目录项。
    * `f2fs_has_inline_xattr(inode) || f2fs_has_inline_dentry(inode)` 检查 inode 是否设置了 `FI_INLINE_XATTR` 或 `FI_INLINE_DENTRY` 标志。
* **`xattr_size = DEFAULT_INLINE_XATTR_ADDRS;`**:  则使用默认的内联 xattr 大小 `DEFAULT_INLINE_XATTR_ADDRS`。
    * `DEFAULT_INLINE_XATTR_ADDRS` 是一个宏，定义了默认的内联 xattr 大小。**这里设置了默认的内联 xattr 大小。** 你需要查找 `DEFAULT_INLINE_XATTR_ADDRS` 的定义来确定具体大小。
* **`fi->i_inline_xattr_size = xattr_size;`**:  将计算得到的 `xattr_size` 赋值给 F2FS inode 信息结构体中的 `i_inline_xattr_size` 字段。**这里最终确定了内联 xattr 的大小，并存储在 `i_inline_xattr_size` 中。**

```c
	fi->i_flags =
		f2fs_mask_flags(mode, F2FS_I(dir)->i_flags & F2FS_FL_INHERITED);
```

* **`fi->i_flags = f2fs_mask_flags(mode, F2FS_I(dir)->i_flags & F2FS_FL_INHERITED);`**: 设置 F2FS inode 信息结构体中的 `i_flags` 字段。
    * `F2FS_I(dir)->i_flags & F2FS_FL_INHERITED` 获取父目录 inode 的 F2FS 特有标志 `i_flags` 中可以被继承的标志 (`F2FS_FL_INHERITED` 定义了哪些标志可以被继承)。
    * `f2fs_mask_flags(mode, ...)`  根据新 inode 的模式 `mode` 和继承的标志，计算出新 inode 的 `i_flags` 值。`f2fs_mask_flags` 函数会根据文件类型 (目录、普通文件等) 和权限，以及继承的标志，设置合适的 F2FS 特有标志。

```c
	if (S_ISDIR(inode->i_mode))
		fi->i_flags |= F2FS_INDEX_FL;
```

* **`if (S_ISDIR(inode->i_mode)) fi->i_flags |= F2FS_INDEX_FL;`**: 如果新 inode 是目录，则设置 `F2FS_INDEX_FL` 标志。
    * `F2FS_INDEX_FL` 标志表示目录需要使用 F2FS 的索引功能来加速目录项查找。

```c
	if (fi->i_flags & F2FS_PROJINHERIT_FL)
		set_inode_flag(inode, FI_PROJ_INHERIT);
```

* **`if (fi->i_flags & F2FS_PROJINHERIT_FL) set_inode_flag(inode, FI_PROJ_INHERIT);`**: 如果 F2FS inode 信息结构体中的 `i_flags` 设置了 `F2FS_PROJINHERIT_FL` 标志 (表示继承项目配额)，则设置 inode 的 `FI_PROJ_INHERIT` 标志。
    * `FI_PROJ_INHERIT` 标志用于在通用 inode 层面标记需要继承项目配额。

```c
	/* Check compression first. */
	set_compress_new_inode(sbi, dir, inode, name);
```

* **`/* Check compression first. */`**: 注释，说明接下来的操作是关于压缩的。
* **`set_compress_new_inode(sbi, dir, inode, name);`**: 设置新 inode 的压缩属性。
    * `set_compress_new_inode(sbi, dir, inode, name)` 是 F2FS 特有的函数，用于根据文件系统选项、父目录和文件名等信息，决定是否对新 inode 启用压缩，并设置相关的压缩参数。**注意这里是在设置压缩属性，而不是直接分配空间或设置大小。压缩的具体实现和空间分配是在后续数据写入时进行的。**

```c
	/* Should enable inline_data after compression set */
	if (test_opt(sbi, INLINE_DATA) && f2fs_may_inline_data(inode))
		set_inode_flag(inode, FI_INLINE_DATA);
```

* **`/* Should enable inline_data after compression set */`**: 注释，说明内联数据 (inline data) 的设置应该在压缩设置之后。
* **`if (test_opt(sbi, INLINE_DATA) && f2fs_may_inline_data(inode))`**: 检查文件系统是否启用了内联数据功能，并且 inode 是否可以内联数据。
    * `test_opt(sbi, INLINE_DATA)` 检查文件系统的选项中是否启用了 `INLINE_DATA` 选项。这个选项允许将小文件的数据直接存储在 inode 结构中，而不是单独分配数据块。
    * `f2fs_may_inline_data(inode)` 是 F2FS 特有的函数，用于判断是否可以将数据内联存储在 inode 中。判断条件可能包括文件类型、文件大小、inode 的可用空间等。
* **`set_inode_flag(inode, FI_INLINE_DATA);`**: 如果条件成立，则设置 inode 的 `FI_INLINE_DATA` 标志。
    * `set_inode_flag(inode, FI_INLINE_DATA)` 设置 inode 的 `i_flags` 字段中的 `FI_INLINE_DATA` 标志，表示这个 inode 可以使用内联数据功能。 **注意，这里只是设置了标志，并没有分配内联数据空间或设置大小。内联数据的空间分配和实际写入是在后续的文件写入操作中进行的。**  `f2fs_may_inline_data` 和 `INLINE_DATA` 选项共同决定了是否启用内联数据，但具体内联数据的大小限制和空间分配机制不在这个函数中直接体现。

```c
	if (name && !test_opt(sbi, DISABLE_EXT_IDENTIFY))
		set_file_temperature(sbi, inode, name);
```

* **`if (name && !test_opt(sbi, DISABLE_EXT_IDENTIFY))`**: 检查是否提供了文件名 `name`，并且文件系统没有禁用扩展标识 (extended identify) 功能。
    * `name` 检查文件名指针是否非空。
    * `!test_opt(sbi, DISABLE_EXT_IDENTIFY)` 检查文件系统的选项中是否禁用了 `DISABLE_EXT_IDENTIFY` 选项。扩展标识功能可能用于根据文件名等信息设置文件的温度 (temperature)，用于分层存储等优化。
* **`set_file_temperature(sbi, inode, name);`**:  根据文件名设置文件的温度。
    * `set_file_temperature(sbi, inode, name)` 是 F2FS 特有的函数，用于根据文件名 `name` 和文件系统选项，设置 inode 的温度属性。

```c
	stat_inc_inline_xattr(inode);
	stat_inc_inline_inode(inode);
	stat_inc_inline_dir(inode);
```

* **`stat_inc_inline_xattr(inode);`**: 增加内联 xattr 相关的统计计数器。
    * `stat_inc_inline_xattr(inode)` 是 F2FS 特有的函数，用于增加内联 xattr 相关的统计信息，例如内联 xattr 的 inode 数量等。
* **`stat_inc_inline_inode(inode);`**: 增加内联 inode 相关的统计计数器。
    * `stat_inc_inline_inode(inode)` 可能是用于统计使用了某些内联特性的 inode 数量，具体含义需要查看代码实现。
* **`stat_inc_inline_dir(inode);`**: 增加内联目录相关的统计计数器。
    * `stat_inc_inline_dir(inode)` 可能是用于统计使用了内联目录项的目录数量。

```c
	f2fs_set_inode_flags(inode);
```

* **`f2fs_set_inode_flags(inode);`**:  设置 F2FS 特有的 inode 标志。
    * `f2fs_set_inode_flags(inode)` 是 F2FS 特有的函数，可能用于根据之前设置的各种标志，最终设置 F2FS inode 信息结构体中的 `i_flags` 字段。

```c
	f2fs_init_extent_tree(inode);
```

* **`f2fs_init_extent_tree(inode);`**: 初始化 inode 的 extent 树。
    * `f2fs_init_extent_tree(inode)` 是 F2FS 特有的函数，用于初始化 inode 的 extent 树。Extent 树用于管理文件的数据块分配信息。

```c
	trace_f2fs_new_inode(inode, 0);
	return inode;
```

* **`trace_f2fs_new_inode(inode, 0);`**:  tracepoint，用于跟踪 `f2fs_new_inode` 函数的成功执行。
    * `trace_f2fs_new_inode(inode, 0)` 是 F2FS 的 tracepoint 宏，用于在内核 tracing 系统中记录 `f2fs_new_inode` 函数的执行信息，`0` 可能表示成功状态。
* **`return inode;`**:  函数成功返回新创建的 inode 指针。

```c
fail:
	trace_f2fs_new_inode(inode, err);
	make_bad_inode(inode);
	if (nid_free)
		set_inode_flag(inode, FI_FREE_NID);
	iput(inode);
	return ERR_PTR(err);
```

* **`fail:`**: 错误处理标签 `fail`。当 inode 创建过程中发生错误，并且错误发生在 inode 编号分配之后，会跳转到这里。
* **`trace_f2fs_new_inode(inode, err);`**: tracepoint，记录 `f2fs_new_inode` 函数的失败执行，并记录错误码 `err`。
* **`make_bad_inode(inode);`**: 将 inode 标记为 "bad"。
    * `make_bad_inode(inode)` 是 Linux 内核提供的函数，用于将 inode 标记为错误状态，防止后续使用。
* **`if (nid_free) set_inode_flag(inode, FI_FREE_NID);`**: 如果 `nid_free` 为真 (表示之前成功分配了 inode 编号)，则设置 inode 的 `FI_FREE_NID` 标志。
    * `FI_FREE_NID` 标志可能用于在 inode 被释放时，释放之前分配的 inode 编号。
* **`iput(inode);`**: 释放 inode 引用计数。
    * `iput(inode)` 是 Linux 内核提供的函数，用于减少 inode 的引用计数。当引用计数降为 0 时，inode 可能会被释放回 inode 缓存。
* **`return ERR_PTR(err);`**: 函数返回错误指针，指示 inode 创建失败，并返回错误码 `err`。

```c
fail_drop:
	trace_f2fs_new_inode(inode, err);
	dquot_drop(inode);
	inode->i_flags |= S_NOQUOTA;
	if (nid_free)
		set_inode_flag(inode, FI_FREE_NIaD);
	clear_nlink(inode);
	unlock_new_inode(inode);
	iput(inode);
	return ERR_PTR(err);
}
```

* **`fail_drop:`**: 错误处理标签 `fail_drop`。当 inode 创建过程中发生错误，并且错误发生在配额初始化之后，会跳转到这里。
* **`trace_f2fs_new_inode(inode, err);`**: tracepoint，记录 `f2fs_new_inode` 函数的失败执行，并记录错误码 `err`。
* **`dquot_drop(inode);`**: 释放 inode 的磁盘配额。
    * `dquot_drop(inode)` 是 F2FS 特有的函数，用于释放之前为 inode 分配的磁盘配额资源。
* **`inode->i_flags |= S_NOQUOTA;`**: 设置 inode 的 `S_NOQUOTA` 标志。
    * `S_NOQUOTA` 标志表示 inode 不再使用配额。
* **`if (nid_free) set_inode_flag(inode, FI_FREE_NIaD);`**: 如果 `nid_free` 为真，则设置 inode 的 `FI_FREE_NIaD` 标志。
    * `FI_FREE_NIaD` 可能是 `FI_FREE_NID` 的变体，具体含义需要查看代码定义。
* **`clear_nlink(inode);`**: 清除 inode 的硬链接计数。
    * `clear_nlink(inode)` 是 Linux 内核提供的函数，用于将 inode 的硬链接计数 `i_nlink` 设置为 0。
* **`unlock_new_inode(inode);`**: 解锁新创建的 inode。
    * `unlock_new_inode(inode)` 是 Linux 内核提供的函数，用于解锁新创建的 inode 上的锁。
* **`iput(inode);`**: 释放 inode 引用计数。
* **`return ERR_PTR(err);`**: 函数返回错误指针，指示 inode 创建失败，并返回错误码 `err`

 好的，我们继续深入分析 `f2fs_new_inode` 函数，并重点关注内联数据、扩展属性以及相关的大小设置。

**总结关键点：内联数据 (Inline Data), 扩展属性 (Xattr), 原子写入 (Atomic Write) 和大小设置**

从逐行解析中，我们可以总结出 `f2fs_new_inode` 函数在创建具有内联数据或扩展属性 inode 时的关键步骤：

1. **内联扩展属性 (Inline Xattr):**
   - **使能条件:**  文件系统需要启用 `INLINE_XATTR` 选项 (`test_opt(sbi, INLINE_XATTR)` 为真)。
   - **标志设置:**  如果满足条件，设置 inode 的 `FI_INLINE_XATTR` 标志 (`set_inode_flag(inode, FI_INLINE_XATTR)`).
   - **大小确定:**
     - **灵活大小:** 如果文件系统支持灵活内联 xattr (`f2fs_sb_has_flexible_inline_xattr(sbi)` 为真)，则内联 xattr 的大小 `xattr_size` 从文件系统选项 `F2FS_OPTION(sbi).inline_xattr_size` 中获取。这意味着管理员可以通过文件系统挂载选项来配置这个大小。
     - **默认大小:**  如果不支持灵活大小，但启用了内联 xattr 或内联 dentry，则使用默认大小 `DEFAULT_INLINE_XATTR_ADDRS`。
   - **大小存储:**  最终确定的 `xattr_size` 会被赋值给 `fi->i_inline_xattr_size`，存储在 F2FS 特有的 inode 信息结构体中。
   - **扩展属性 (Extra Attribute):**  如果文件系统支持扩展属性功能 (`f2fs_sb_has_extra_attr(sbi)` 为真)，则会设置 `FI_EXTRA_ATTR` 标志，并且 `fi->i_extra_isize` 会被设置为 `F2FS_TOTAL_EXTRA_ATTR_SIZE`，这是一个预定义好的固定大小，用于存储更大的扩展属性数据。

2. **内联数据 (Inline Data):**
   - **使能条件:** 文件系统需要启用 `INLINE_DATA` 选项 (`test_opt(sbi, INLINE_DATA)` 为真)，并且 `f2fs_may_inline_data(inode)` 函数需要返回真，表示当前 inode 适合使用内联数据。`f2fs_may_inline_data` 的具体判断逻辑可能涉及到文件类型、大小限制等，这部分逻辑不在 `f2fs_new_inode` 函数中，而是在 `f2fs_may_inline_data` 函数的实现中。
   - **标志设置:** 如果满足条件，设置 inode 的 `FI_INLINE_DATA` 标志 (`set_inode_flag(inode, FI_INLINE_DATA)`).
   - **大小设置:**  **注意，在 `f2fs_new_inode` 函数中，并没有显式地设置内联数据的大小或分配内联数据空间。**  `FI_INLINE_DATA` 标志仅仅是标记了这个 inode *可以* 使用内联数据功能。 实际的内联数据空间分配和数据写入操作，以及内联数据大小的限制，会在后续的文件写入操作中处理。

3. **原子写入 (Atomic Write):**
   - 函数本身没有直接提到 "原子写入"。
   - 然而，内联数据和内联 xattr 这些特性，**在一定程度上可以支持小文件的原子写入。**  因为对于足够小的文件，其数据和元数据可以完全存储在 inode 结构内部的内联区域，这样对 inode 的更新操作（例如写入小文件）就可以在一次操作中完成，从而实现原子性。
   - F2FS 的原子写入能力更复杂，可能还涉及到日志 (logging) 或其他机制，`f2fs_new_inode` 函数只是为原子写入提供了基础，即通过内联特性为小文件的数据和元数据提供了一个可能的原子更新单元。

**关于大小定义的查找:**

为了确定 `F2FS_TOTAL_EXTRA_ATTR_SIZE`, `F2FS_OPTION(sbi).inline_xattr_size`, `DEFAULT_INLINE_XATTR_ADDRS` 这些宏和选项的具体数值，你需要查阅 F2FS 的源代码，特别是以下文件：

- **`f2fs.h` 或 `f2fs_fs.h`**:  通常会定义 `F2FS_TOTAL_EXTRA_ATTR_SIZE` 和 `DEFAULT_INLINE_XATTR_ADDRS` 这样的宏。
- **`f2fs_sb.h` 或 `f2fs_options.h`**:  可能会定义 `F2FS_OPTION` 结构体，以及 `inline_xattr_size` 字段。
- **Kconfig 文件 (在 Linux 内核源代码树中搜索 `F2FS_` 相关的 Kconfig 选项)**:  可以找到 `INLINE_XATTR`, `INLINE_DATA` 等文件系统选项的默认值和配置方法。

通过 `grep` 命令在内核源代码树中搜索这些宏和选项的定义，或者使用内核代码交叉引用工具 (如 lxr.linux.no 或 sourcegraph) 可以快速找到它们的具体数值和定义位置。

**总结 `f2fs_new_inode` 的作用:**

总而言之，`f2fs_new_inode` 函数的主要职责是：

- **分配和初始化 inode 结构:**  包括通用 inode 结构 `inode` 和 F2FS 特有的 inode 信息结构 `f2fs_inode_info`。
- **分配 inode 编号 (NID).**
- **设置 inode 的基本属性:**  例如所有者、权限、时间戳、inode 编号等。
- **设置 F2FS 特有的 inode 标志:**  例如 `FI_INLINE_XATTR`, `FI_INLINE_DATA`, `FI_EXTRA_ATTR` 等，这些标志控制了 inode 是否启用内联数据、扩展属性等功能。
- **初始化 inode 的数据结构:** 例如 extent 树。
- **处理配额和加密等功能.**

**重要提示:**  `f2fs_new_inode` 函数主要负责 inode 的 *创建和初始化*，以及 *设置各种标志*。  对于内联数据和扩展属性，这个函数只负责 *标记 inode 是否可以使用这些功能* 以及 *设置一些相关的大小参数 (例如内联 xattr 的大小)*。  **实际的内联数据空间分配、数据写入、以及原子写入的具体实现，是在文件系统的其他操作函数中完成的，例如 `f2fs_write_data_page`, `f2fs_setxattr` 等。**