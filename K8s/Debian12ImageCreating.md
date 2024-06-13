# Debian 12 镜像制作
## 1 创建项目结构
```
mkdir debian12-scratch
cd debian12-scratch
```
## 2 获取Debian 12的rootfs
Debian官方提供基础的rootfs文件，可以从他们的FTP或HTTP服务器上下载。我讲使用`debootstrap`来生成一个最小的Debian 12 根文件系统。
```
sudo apt install debootstrap
mkdir debian-rootfs
sudo debootstrap --arch=amd64 bookworm debian-rootfs http://deb.debian.org/debian
```
- `--arch=amd64`：指定架构为 64 位。
- `bookworm`：`Debian 12` 的代号。
- `debian-rootfs`：目标目录。
- `http://deb.debian.org/debian`：`Debian` 仓库地址。

  
