

在 Bash 脚本中，可以使用 `test` 命令或者其简写形式 `[` 来进行文件测试。下面列出了一些常用的文件测试选项及其含义：

- **-b FILE**  
  测试 FILE 是否存在且为块设备文件。

- **-c FILE**  
  测试 FILE 是否存在且为字符设备文件。

- **-d FILE**  
  测试 FILE 是否存在且为目录。

- **-e FILE**  
  测试 FILE 是否存在。（注意：虽然 POSIX 标准中没有规定 `-e`，但 GNU Bash 中可用）

- **-f FILE**  
  测试 FILE 是否存在且为普通文件。

- **-g FILE**  
  测试 FILE 是否存在且设置了 set-group-ID 标志。

- **-h FILE** 或 **-L FILE**  
  测试 FILE 是否存在且为符号链接。

- **-p FILE**  
  测试 FILE 是否存在且为命名管道（FIFO）。

- **-r FILE**  
  测试 FILE 是否存在且当前用户具有读取权限。

- **-s FILE**  
  测试 FILE 是否存在且大小大于零。

- **-S FILE**  
  测试 FILE 是否存在且为套接字（socket）。

- **-t FD**  
  测试文件描述符 FD 是否打开并且指向终端设备。（通常用来判断脚本是否在交互式终端中运行，如 `-t 0` 判断标准输入）

- **-u FILE**  
  测试 FILE 是否存在且设置了 set-user-ID 标志。

- **-w FILE**  
  测试 FILE 是否存在且当前用户具有写入权限。

- **-x FILE**  
  测试 FILE 是否存在且当前用户具有执行权限。

- **-O FILE**  
  测试 FILE 是否存在且文件所有者为当前有效用户 ID。

- **-G FILE**  
  测试 FILE 是否存在且文件所属组为当前有效组 ID。

- **-N FILE**  
  测试 FILE 是否存在且自上次读取后已被修改。（部分系统支持）

此外，还有一些用于比较两个文件时间戳或判断是否为同一文件的测试选项：

- **FILE1 -nt FILE2**  
  如果 FILE1 比 FILE2 新，则返回真。

- **FILE1 -ot FILE2**  
  如果 FILE1 比 FILE2 旧，则返回真。

- **FILE1 -ef FILE2**  
  如果 FILE1 和 FILE2 指向同一设备上的同一 inode（即为同一文件），则返回真。

这些选项可以在 Bash 脚本中用来根据文件的状态做出不同的逻辑判断。使用示例：

```bash
if [ -f "/path/to/file" ]; then
    echo "文件存在且为普通文件"
fi

if [ -s "/path/to/file" ]; then
    echo "文件存在且非空"
fi
```

这样就能根据文件的不同属性来编写相应的脚本逻辑。
