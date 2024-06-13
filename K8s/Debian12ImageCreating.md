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

## 3 创建dockerfile
```
# 使用 scratch 作为基础镜像
FROM scratch

# 复制根文件系统
COPY debian-rootfs/ /

# 设置环境变量
ENV PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

# 删除不必要的包缓存
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# 设置默认命令
CMD ["/bin/bash"]
```
- `FROM scratch`：使用 `scratch` 作为基础镜像。
- `COPY debian-rootfs/ /`：将根文件系统复制到镜像中。
- `ENV PATH`：设置默认的 PATH 环境变量。
- `CMD ["/bin/bash"]`：设置默认启动命令。

## 4 构建镜像
```
docker build -t debian12-scratch .
```
## 5 运行Docker镜像
运行构建好的镜像，测试其功能。
```
docker run -it debian12-scratch
```
你应该能够进入一个基于 Debian 12 的 shell 环境。如果一切正常，你可以进行进一步的定制和优化以及部署自己的业务软件。
