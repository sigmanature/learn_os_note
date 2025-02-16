## 用户:
有办法把qemu启动时挂载的文件系统同时挂载到宿主机的某个目录使得宿主机对该目录的修改能直接反映到qemu系统的这个目录吗?

## ChatGPT:
是的，可以通过 **共享目录（Shared Folder）** 或者 **NFS（Network File System）** 实现 **宿主机和 QEMU 虚拟机的文件同步**，这样你在宿主机修改的文件能立即在 QEMU 虚拟机里生效。

---

## **方法 1：使用 9p Virtio 共享目录**
**9p Virtio** 是 QEMU 内置的一种共享文件系统方式，它允许宿主机的某个目录直接映射到 QEMU 虚拟机的目录中，实现 **实时文件同步**。

### **步骤 1：在宿主机上创建共享目录**
假设你想共享 `/home/user/shared` 目录，先确保它存在：
```sh
mkdir -p /home/user/shared
```

### **步骤 2：启动 QEMU 时添加 `-virtfs` 参数**
启动 QEMU 时，在命令行添加：
```sh
qemu-system-aarch64 \
    -drive file=ubuntu.img,if=virtio \
    -virtfs local,path=/home/user/shared,mount_tag=hostshare,security_model=passthrough,id=hostshare \
    ...
```
解释：
- `path=/home/user/shared`：宿主机上的共享目录路径。
- `mount_tag=hostshare`：挂载点名称（虚拟机里要用这个名称挂载）。
- `security_model=passthrough`：让虚拟机能直接访问宿主机文件的权限。

### **步骤 3：在 QEMU 里挂载共享目录**
进入 QEMU 虚拟机后，运行：
```sh
sudo mount -t 9p -o trans=virtio hostshare /mnt
```
如果一切正常，`/mnt` 目录现在就会映射到宿主机的 `/home/user/shared` 目录。

你可以试试：
```sh
echo "Hello from Host" > /home/user/shared/test.txt
ls /mnt
cat /mnt/test.txt  # 在 QEMU 里读取
```
这样，宿主机对 `/home/user/shared` 的修改会立即反映到 QEMU 虚拟机的 `/mnt` 目录。

---

## **方法 2：使用 NFS 共享**
如果 9p 方式在 ARM64 下不稳定，可以使用 **NFS（Network File System）**。

### **步骤 1：在宿主机上安装 NFS 服务器**
```sh
sudo apt update
sudo apt install -y nfs-kernel-server
```
然后创建一个共享目录：
```sh
mkdir -p /home/user/shared
sudo chmod 777 /home/user/shared
```

编辑 `/etc/exports`，添加：
```
/home/user/shared *(rw,sync,no_subtree_check,no_root_squash)
```
然后重启 NFS 服务：
```sh
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
```

### **步骤 2：在 QEMU 里挂载 NFS**
在 QEMU 里安装 NFS 客户端：
```sh
sudo apt update
sudo apt install -y nfs-common
```
然后挂载：
```sh
sudo mount -t nfs <宿主机IP>:/home/user/shared /mnt
```
（把 `<宿主机IP>` 换成你的宿主机 IP，比如 `192.168.1.100`。）

这样，QEMU 里的 `/mnt` 目录就会映射到宿主机的 `/home/user/shared`，文件能 **实时同步**。

---

## **哪种方法更好？**
| 方案 | 优势 | 适用情况 |
|------|------|---------|
| **9p Virtio** | 配置简单，直接用 QEMU 自带的功能 | 适用于本地开发，适用于 QEMU 虚拟机 |
| **NFS 共享** | 性能更好，支持多个虚拟机同时访问 | 适用于远程访问或多个虚拟机共享 |

如果你只是想让宿主机和 QEMU 共享一个目录，**9p Virtio** 是最简单的方式；如果需要更高性能或者多个 QEMU 共享目录，**NFS** 会更稳定。

你可以试试 9p Virtio，如果有问题，我可以帮你调试！

> provided by [EasyChat](https://site.eqing.tech/)

## 用户:
我先看第一种方式啊
#! /bin/bash
qemu-system-aarch64 \
    -smp 8 \
    -machine virt,virtualization=true,gic-version=3 \
    -nographic \
    -m size=4G \
    -cpu cortex-a72 \
    -kernel ./linux-6.13.1/arch/arm64/boot/Image \
    -netdev user,id=eth0,hostfwd=tcp::5022-:22,hostfwd=tcp::5080-:80 -device virtio-net-device,netdev=eth0 \
    -drive format=raw,file=ubuntu.img \
    -append "noinitrd root=/dev/vda rw console=ttyAMA0 nokaslr loglevel=8 " \
    -s \
我原先的启动脚本是这样的
然后呢首先我问问-drive里的if=virtio是什么意思
然后-virtfs 本身是什么意思

## ChatGPT:
### **1. `-drive` 参数中 `if=virtio` 的含义**
`-drive` 参数用于指定虚拟机的磁盘配置，而 `if=virtio` 是指定该磁盘的 **接口类型**。

- **`if=virtio`** 指的是 **Virtio** 设备接口。Virtio 是一种用于虚拟化环境的高性能 I/O 虚拟化接口，主要目的是提高虚拟机内存和磁盘 I/O 性能。
- **为什么用 `if=virtio`？**
  - 在虚拟机中，Virtio 提供比传统 IDE 或 SCSI 模拟更高效的 I/O 操作。
  - 如果你不指定 `if=virtio`，QEMU 可能默认使用较慢的 IDE 接口，这会降低虚拟机的性能。

例如：
```sh
-drive file=ubuntu.img,if=virtio
```
- 表示 QEMU 使用 Virtio 磁盘接口来访问 `ubuntu.img` 文件。
- 在虚拟机内部，你的磁盘设备可能会显示为 `/dev/vda`，而不是传统 IDE 的 `/dev/sda`。

---

### **2. `-virtfs` 参数的作用**
`-virtfs` 是 QEMU 用于 **共享宿主机目录到虚拟机** 的参数，它实现了类似网络共享的效果，可以让宿主机的目录映射到虚拟机内的指定路径。

#### **基本格式**
```sh
-virtfs local,path=<宿主机目录>,mount_tag=<标签>,security_model=<模式>,id=<ID>
```
- **`local`**：表示共享的是本地目录。
- **`path`**：宿主机上要共享的目录路径。
- **`mount_tag`**：一个挂载标签，虚拟机中挂载时需要用到。
- **`security_model`**：
  - **`passthrough`**：虚拟机使用宿主机文件的原始权限。
  - **`mapped-xattr`**：映射宿主机用户权限到虚拟机。
- **`id`**：用来标识共享目录的唯一 ID。

#### **示例解释**
```sh
-virtfs local,path=/home/user/shared,mount_tag=hostshare,security_model=passthrough,id=hostshare
```
- **宿主机 `/home/user/shared` 目录** 将被映射到虚拟机中。
- **挂载时使用标签 `hostshare`** 识别。
- **`security_model=passthrough`**：虚拟机能直接以宿主机文件权限操作。

#### **在 QEMU 虚拟机中挂载**
进入虚拟机后，运行：
```sh
sudo mount -t 9p -o trans=virtio hostshare /mnt
```
- **`-t 9p`**：使用 9p 文件系统协议。
- **`trans=virtio`**：通过 Virtio 传输数据。
- **`hostshare`**：是宿主机定义的挂载标签。

这样，虚拟机中的 `/mnt` 目录就可以直接访问宿主机的 `/home/user/shared`。

---

### **3. 修改你的 QEMU 启动脚本**
在现有脚本中加入 `-virtfs`：
```bash
qemu-system-aarch64 \
    -smp 8 \
    -machine virt,virtualization=true,gic-version=3 \
    -nographic \
    -m size=4G \
    -cpu cortex-a72 \
    -kernel ./linux-6.13.1/arch/arm64/boot/Image \
    -netdev user,id=eth0,hostfwd=tcp::5022-:22,hostfwd=tcp::5080-:80 -device virtio-net-device,netdev=eth0 \
    -drive format=raw,file=ubuntu.img,if=virtio \
    -virtfs local,path=/home/user/shared,mount_tag=hostshare,security_model=passthrough,id=hostshare \
    -append "noinitrd root=/dev/vda rw console=ttyAMA0 nokaslr loglevel=8 " \
    -s
```

### **4. 测试共享目录**
启动虚拟机后，在虚拟机中运行：
```sh
sudo mount -t 9p -o trans=virtio hostshare /mnt
```
然后，你可以：
```sh
echo "Test from host" > /home/user/shared/test.txt
cat /mnt/test.txt  # 在虚拟机中查看
```
修改 `/home/user/shared` 中的文件会直接反映在虚拟机的 `/mnt` 中，反之亦然。

---

### **总结**
- **`-drive if=virtio`** 提高了磁盘性能，建议使用。
- **`-virtfs`** 提供了简单高效的宿主机和虚拟机共享目录的方案。
- 共享目录可用于快速在宿主机上开发、在虚拟机中实时测试。

> provided by [EasyChat](https://site.eqing.tech/)

## 用户:
sudo mount -t 9p -o trans=virtio hostshare /mnt
每次系统关机的时候 所有挂载点会卸载是吧
那么我应该怎么样才能使得系统启动的时候自动进行挂载呢?

## ChatGPT:
是的，每次 **QEMU 虚拟机关机后，挂载点会被卸载**，你需要在系统启动时 **自动挂载** 共享目录。可以通过 **`/etc/fstab`** 或 **`systemd`** 来实现。

---

## **方法 1：修改 `/etc/fstab`（推荐）**
你可以在 QEMU 虚拟机的 `/etc/fstab` 文件中添加 9p 挂载配置，让它在每次启动时自动挂载。

### **步骤 1：编辑 `/etc/fstab`**
在 QEMU 虚拟机中，运行：
```sh
sudo nano /etc/fstab
```
在文件末尾添加：
```ini
hostshare  /mnt  9p  trans=virtio,version=9p2000.L  0  0
```
**解释：**
- **`hostshare`**：挂载标签，与 `-virtfs` 中的 `mount_tag=hostshare` 一致。
- **`/mnt`**：挂载到 `/mnt` 目录。
- **`9p`**：使用 9p 文件系统。
- **`trans=virtio`**：使用 Virtio 传输方式。
- **`version=9p2000.L`**：建议使用 `9p2000.L` 版本，兼容性更好。

### **步骤 2：测试**
运行：
```sh
sudo mount -a
```
如果没有报错，说明 `/etc/fstab` 配置正确，**下次系统启动时会自动挂载**。

---

## **方法 2：使用 systemd 自动挂载**
如果 `/etc/fstab` 不起作用，可以使用 `systemd` 添加 **开机自动挂载任务**。

### **步骤 1：创建 `systemd` 挂载服务**
在 QEMU 虚拟机里，运行：
```sh
sudo nano /etc/systemd/system/mnt-share.mount
```
写入以下内容：
```ini
[Unit]
Description=Mount QEMU 9p Shared Folder
After=network.target

[Mount]
What=hostshare
Where=/mnt
Type=9p
Options=trans=virtio,version=9p2000.L

[Install]
WantedBy=multi-user.target
```

### **步骤 2：启用并启动服务**
运行：
```sh
sudo systemctl daemon-reload
sudo systemctl enable mnt-share.mount
sudo systemctl start mnt-share.mount
```
**这样，QEMU 启动时 `/mnt` 目录会自动挂载共享目录**。

---

## **总结**
| 方法 | 适用情况 | 配置方式 |
|------|---------|---------|
| **`/etc/fstab`（推荐）** | 适用于大多数 Linux 发行版 | 编辑 `/etc/fstab` 并添加 `9p` 挂载 |
| **`systemd` 挂载** | `/etc/fstab` 方式不生效时 | 创建 `mnt-share.mount` 并 `enable` |

你可以先试 `/etc/fstab`，如果不行再用 `systemd`。试试看，如果有问题我可以帮你调试！

> provided by [EasyChat](https://site.eqing.tech/)
