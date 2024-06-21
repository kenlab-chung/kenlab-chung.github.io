# 离线安装Kubernetes
## 1 前言
操作系统：debian 12
## 1 准备工作
### 1.1 下载K8S依赖镜像
-在互联网机器上下载K8S依赖镜像
```
docker pull registry.aliyuncs.com/google_containers/kube-apiserver:v1.30.0
docker pull registry.aliyuncs.com/google_containers/kube-controller-manager:v1.30.0
docker pull registry.aliyuncs.com/google_containers/kube-scheduler:v1.30.0
docker pull registry.aliyuncs.com/google_containers/kube-proxy:v1.30.0
docker pull registry.aliyuncs.com/google_containers/coredns:1.11.1
docker pull registry.aliyuncs.com/google_containers/pause:3.9
docker pull registry.aliyuncs.com/google_containers/etcd:3.5.12-0
```
- 将docker镜像保存存为tar包，供离线使用
```
docker save -o kube-apiserver-v1.30.0.tar registry.aliyuncs.com/google_containers/kube-apiserver:v1.30.0
docker save -o kube-controller-manager-v1.30.0.tar registry.aliyuncs.com/google_containers/kube-controller-manager:v1.30.0
docker save -o kube-scheduler-v1.30.0.tar registry.aliyuncs.com/google_containers/kube-scheduler:v1.30.0
docker save -o kube-proxy-v1.30.0.tar registry.aliyuncs.com/google_containers/kube-proxy:v1.30.0
docker save -o coredns-1.11.1.tar registry.aliyuncs.com/google_containers/coredns:1.11.1
docker save -o pause-3.9.tar registry.aliyuncs.com/google_containers/pause:3.9
docker save -o etcd-3.5.12-0.tar registry.aliyuncs.com/google_containers/etcd:3.5.12-0
```
### 1.2 下载registry
下载registry镜像
```
docker pull docker.io/registry
docker save -o docker-registry.tar  docker.io/registry
```
安装docker-registry
```
# 解压镜像
docker load -i docker-registry.tar
# 运行docker-registry
mkdir -p /opt/software/registry-data
docker run -d --name registry --restart=always -v /opt/software/registry-data:/var/lib/registry -p 5000:5000 docker.io/registry

查看是否已运行
# docker ps
CONTAINER ID   IMAGE          COMMAND                   CREATED          STATUS          PORTS                                                                                                                     NAMES
72b1ee0dd35d   registry       "/entrypoint.sh /etc…"   17 seconds ago   Up 15 seconds   0.0.0.0:5000->5000/tcp, :::5000->5000/tcp                                                                                     registry
```
