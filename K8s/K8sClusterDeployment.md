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
