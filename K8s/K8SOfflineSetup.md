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
### 1.2 部署registry
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
## 2 部署K8S
将K8S依赖的镜像上传至k8s-master01节点
```
docker load -i kube-apiserver-v1.30.0.tar
docker load -i kube-controller-manager-v1.30.0.tar
docker load -i kube-scheduler-v1.30.0.tar
docker load -i kube-proxy-v1.30.0.tar
docker load -i coredns-1.11.1.tar
docker load -i pause-3.9.tar

docker tag registry.aliyuncs.com/google_containers/kube-apiserver:v1.30.0 192.168.115.11:81/kube-apiserver:v1.30.0
docker tag registry.aliyuncs.com/google_containers/kube-controller-manager:v1.30.0 192.168.115.11:81/kube-controller-manager:v1.30.0
docker tag registry.aliyuncs.com/google_containers/kube-scheduler:v1.30.0 192.168.115.11:81/kube-scheduler:v1.30.0
docker tag registry.aliyuncs.com/google_containers/kube-proxy:v1.30.0 192.168.115.11:81/kube-proxy:v1.30.0
docker tag registry.aliyuncs.com/google_containers/coredns:1.11.1 192.168.115.11:81/coredns:v1.11.1
docker tag registry.aliyuncs.com/google_containers/pause:3.9 192.168.115.11:81/pause:3.9
```
在每台机器执行配置，将docker-registry以及k8s的镜像的地址配置到/etc/docker/daemon.json中
```
Vi /etc/docker/daemon.json
添加配置
"insecure-registries":["192.168.115.11:81", "quay.io", "k8s.gcr.io", "gcr.io"]

[root@k8s-master02 ~]# cat /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "insecure-registries":["192.168.115.11:5000", "quay.io", "k8s.gcr.io", "gcr.io"]
}

# 重启docker
sytemctl daemon-reload
systemctl restart docker
```
在k8s-master01上将镜像推送到docker-registry
```
docker push 192.168.115.11:81/kube-apiserver:v1.30.0
docker push 192.168.115.11:81/kube-controller-manager:v1.30.0
docker push 192.168.115.11:81/kube-scheduler:v1.30.0
docker push 192.168.115.11:81/kube-proxy:v1.30.0
docker push 192.168.115.11:81/coredns:v1.11.1
docker push 192.168.115.11:81/pause:3.9
```
修改cri-docker将pause镜像修改为docker-registry中的配置(每台电脑都执行)
```
# vi /usr/lib/systemd/system/cri-docker.service
# 修改--pod-infra-container-image=registry.k8s.io/pause:3.9 为--pod-infra-container-image=192.168.115.11:81/pause:3.9
# 重启cri-docker
systemctl daemon-reload
systemctl restart cri-docker
```
