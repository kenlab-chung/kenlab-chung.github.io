# K8s集群部署指南
## 1 环境

|       IP       |   操作系统  |   主机名称  |       配置        |    网络 |
|----------------|-------------|-------------|-------------------|---------|
|  192.168.1.110 |  CentOS7.9  |  k8s-master | 4vCPU/2G内存/20G硬 | 桥接模式 |
|  192.168.1.111 |  CentOS7.9  |  k8s-node01 | 4vCPU/2G内存/20G硬 | 桥接模式 |
|  192.168.1.112 |  CentOS7.9  |  k8s-node02 | 4vCPU/2G内存/20G硬 | 桥接模式 |

## 2 操作系统配置
### 2.1 分别修改三台主机名称
```
# 设置 192.168.1.110 主机名
hostnamectl set-hostname  k8s-master
# 设置 192.168.1.111 主机名
hostnamectl set-hostname  k8s-node01
# 设置 192.168.1.112 主机名
hostnamectl set-hostname  k8s-node02
```
### 2.2 配置hosts文件(所有节点)
```
cat >> /etc/hosts << EOF
192.168.1.110 k8s-master
192.168.1.111 k8s-node01
192.168.1.112 k8s-node02
EOF
```
### 2.3 修改yum源(所有节点)
```
# 备份原来的yum源
mv /etc/yum.repos.d/CentOS-Base.repo  /etc/yum.repos.d/CentOS-Base.repo.backup

# 下载阿里yum源
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

# 配置安装 K8s 需要的 yum 源
cat <<EOF >/etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 清理yum源
yum clean all

# 生存新的yum缓存
yum makecache fast

# 更新yum源
yum -y update
```
### 2.4 关闭防火墙及防火墙自启动（所有节点）
```
systemctl stop firewalld && systemctl disable firewalld
```
### 2.4 时间同步（所有节点）
```
yum install ntpdate -y
ntpdate time.windows.com
```
### 2.5 关闭selinux（所有节点）
重启才能生效
```
# 永久
sed -i 's/enforcing/disabled/' /etc/selinux/config
# 临时
setenforce 0
# 查看状态
sestatus
```
### 2.6 关闭swap（所有节点）
```
# 临时
swapoff -a
# 永久
vi /etc/fstab
//注释或删除swap的行 
#/dev/mapper/cs-swap     none                    swap    defaults        0 0
# 查看是否关闭
free -h
```
### 2.7 修改内核参数（所有节点）
```
# 将桥接的IPv4流量传递到iptables的链
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
# 生效
sysctl --system
```
### 2.8 服务器免登录（k8s-master节点)
```
# 生成秘钥
ssh-keygen -t rsa
进行三次回车
# 通过 scp 把生成的 id_rsa.pub（公钥）内容复制到目标主机中的 /root/.ssh/authorized_keys 文件下
ssh root@192.168.1.111 ls -ld /root/.ssh/
ssh root@192.168.1.111 mkdir -p /root/.ssh/
scp -p ~/.ssh/id_rsa.pub root@192.168.1.111:/root/.ssh/authorized_keys
ssh root@192.168.1.112 ls -ld /root/.ssh/
ssh root@192.168.1.112 mkdir -p /root/.ssh/
scp -p ~/.ssh/id_rsa.pub root@192.168.1.112:/root/.ssh/authorized_keys
# 服务器上进行免密连接测试
ssh root@192.168.1.112
```
## 3 安装Docker（所有节点）
```
# 获取 Docker yum源
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
# 查看 Docker 版本
yum list docker-ce --showduplicates | sort -r
# 安装 Docker
yum -y install docker-ce-18.06.3.ce-3.el7
# 修改 docker 配置文件
sudo mkdir -p /etc/docker/
sudo touch /etc/docker/daemon.json
cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
  "overlay2.override_kernel_check=true"
  ]
}
EOF
# 重启docker服务
systemctl daemon-reload && systemctl enable docker && systemctl restart docker
```
##  4 开启bridge模式(所有节点)
```
# 临时
echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables
echo 1 > /proc/sys/net/bridge/bridge-nf-call-ip6tables
# 永久
echo """
vm.swappiness = 0
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
""" > /etc/sysctl.conf
# 配置生效
sysctl -p
```
## 5 开启IPVS
官方推荐开通ipvs内核（所有节点）。原因：k8s工作时会用的数据包转发，如果不开启ipvs将会使用iptables转发数据包，而iptables效率很低。
```
# 配置
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
ipvs_modules="ip_vs ip_vs_lc ip_vs_wlc ip_vs_rr ip_vs_wrr ip_vs_lblc ip_vs_lblcr ip_vs_dh ip_vs_sh ip_vs_nq ip_vs_sed ip_vs_ftp nf_conntrack"
for kernel_module in \${ipvs_modules}; do
 /sbin/modinfo -F filename \${kernel_module} > /dev/null 2>&1
 if [ $? -eq 0 ]; then
 /sbin/modprobe \${kernel_module}
 fi
done
EOF
# 变成可执行文件，让这个配置生效
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep ip_vs
```
## 6 部署K8s集群
目前生产环境部署Kubernetes集群主要有两种方式：

- Kubeadm：Kubeadm是一个K8s部署工具，提供Kubeadm init和Kubeadm join，用于快速部署Kubernetes集群。
- 二进制：从github上下载发现版本的二进制包，手动部署每个组件，组成Kubernetes集群。

官方地址：
```
https://kubernetes.io/docs/reference/setup-tools/kubeadm/
```
本文使用Kubeadm方式搭建集群，但建议在生产环境部署Kubernetes集群是使用二进制方式。因为Kubeadm快速帮我们搭建好集群后，不容易发现可能存在的问题，比如配置问题等。
### 6.1 安装Kubeadm、Kubelet、Kubectl工具
由于版本更新频繁，这里指定版本号部署（所有节点）。
```
# 安装
yum install -y kubelet-1.23.0 kubeadm-1.23.0 kubectl-1.23.0
# 安装完成以后不要启动，设置开机自启动即可
systemctl enable kubelet
```
### 6.2 初始化k8s集群
注意api-server的IP必须指定为master的IP（master节点）
```
kubeadm init \
  --apiserver-advertise-address=192.168.1.110 \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.23.0 \
  --service-cidr=10.96.0.0/12 \
  --pod-network-cidr=10.244.0.0/16\
  --ignore-preflight-errors=all
```
|       参数                   |   说明                                                          |   
|------------------------------|----------------------------------------------------------------|
|  apiserver-advertise-address |  指定Master的那个IP地址与其他节点通信                             |  
|  image-repository            |  由于默认拉取地址k8s.gcr.io国内无法访问，这里指定阿里云镜像仓库地址 |  
|   kubernetes-version         |   k8s版本，与上面安装的版本一致，使用命令查看：rpm -q kubeadm      |  
|   service-cidr               |  集群内部虚拟网络，Pod统一访问入口                                | 
|   pod-network-cidr           |   Pod网络，与下面部署的CNI网络组件yaml中保持一致                   | 
|   ignore-prefight-errors     |   忽视警告错误，加上--ignore-prefight-errors=all参数即可          | 
执行Kubeadm init成功执行，控制台输出如下：
```
[init] Using Kubernetes version: v1.23.0
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8s-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.1.110]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [k8s-master localhost] and IPs [192.168.1.110 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [k8s-master localhost] and IPs [192.168.1.110 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 14.007097 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.23" in namespace kube-system with the configuration for the kubelets in the cluster
NOTE: The "kubelet-config-1.23" naming of the kubelet ConfigMap is deprecated. Once the UnversionedKubeletConfigMap feature gate graduates to Beta the default name will become just "kubelet-config". Kubeadm upgrade will handle this transition transparently.
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node k8s-master as control-plane by adding the labels: [node-role.kubernetes.io/master(deprecated) node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node k8s-master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: rawp9y.0223try0okckj9kx
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.1.110:6443 --token rawp9y.0223try0okckj9kx \
        --discovery-token-ca-cert-hash sha256:de7f72ae8edd7bdbaa66f3bd168a80cee4f3aaf3c2c929ece55160265e9ca7ea
```
 根据日志，可以查看初始化k8s集群时需要做哪些工作：
```
[init]：指定版本进行初始化操作。
[preflight]：初始化前的检查和下载所需要的Docker镜像文件。
[kubelet-start]：生成kubelet配置文件/var/lib/kubelet/config.yaml，没有这个文件kubelet无法启动，所以初始化之前启动kubelet，实际上是启动失败的。
[certificates]：生成Kubernetes证书，存放在/etc/kubernetes/pki目录中。
[kubeconfig]：生成KubeConfig文件，存放在/etc/kubernetes目录中，组件之间通信需要使用对应文件。
[control-plane]：使用/etc/kubernetes/manifests目前下了YAML文件，安装Master组件。
[etcd]：使用/etc/kubernetes/manifests/etcd.yaml文件安装Etcd服务。
[wait-control-plane]：等待control-plan部署的Master组件启动。
[apiclient]：检查Master组件服务状态。
[upload-config]：更新配置。
[kubelet]：使用configMap配置kubelet。
[patchnode]：更新CNI信息到Node上，通过注释的方式记录。
[mark-control-plane]：为当前节点打标签，打了角色Master和不可调度标签，这样默认情况下就不会使用Master节点来运行Pod。
[bootstrap-token]：生成token记录下来，后面使用kubeadm join往集群添加节点时会用到。
[addons]：安装附加组件CoreDNS和kube-proxy。
```
### 6.3 创建kube目录，添加kubectl配置
执行节点：master节点
```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```
### 6.4 配置Pod网络组建
使用命令`kubectl get nodes `查看Master的状态是否未就绪，即NotReady状态。
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/1545e80a-7047-4a0f-99b8-fb0664e8e38b)
之所以是这种状态的原因是还缺少一个附件flannel或者Calico，没有网络Pod是无法通信的，所以执行以下命令下载kube-flannel.yml文件
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
