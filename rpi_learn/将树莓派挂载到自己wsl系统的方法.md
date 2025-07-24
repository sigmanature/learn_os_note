### 1.挂载命令
```powershell
# 仍在“以管理员身份运行”的 PowerShell ⬇︎
wsl --mount \\.\PHYSICALDRIVE1 --partition 2 --type ext4
# 这个命令基于的前提假设是我的树莓派总是第二个nvme盘并且啊从分区来说的话总是第二个分区是我的根文件系统
```
### 2.进入挂载的目录
```powershell
# 这只是一个示范和示例
wsl -u root                   # 直接以 root 进入默认发行版（Ubuntu 等）
cd /mnt/wsl/PHYSICALDRIVE1/part2
```
### 3.退出和解除挂载
```powershell
wsl --unmount \\.\PHYSICALDRIVE1
```
