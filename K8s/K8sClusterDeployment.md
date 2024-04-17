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
master节点
```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```
### 6.4 配置Pod网络组建
master节点
使用命令`kubectl get nodes `查看Master的状态是否未就绪，即NotReady状态。

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/1545e80a-7047-4a0f-99b8-fb0664e8e38b)

之所以是这种状态的原因是还缺少一个附件flannel或者Calico，没有网络Pod是无法通信的，所以执行以下命令下载kube-flannel.yml文件
```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
如果文件下载错误，则将kube-flannel.yml手动下载文件到本地。再次执行kubectl命令：
```
kubectl apply -f kube-flannel.yml
```
执行命令后，可以看到很多组件被创建出来了。

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/b1b03837-02d5-4f67-812b-71adab168242)

此时需要查看flannel是否处于正常启动和运行状态，只有组件处于这个状态时部署才算完成。
首先查看flannel镜像是否被拉取下来。从下图可以看到flannel镜像已经被拉取到（需要等待1-2分钟左右）：

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/cea67402-c12f-49da-99ee-3dc86b54d7b8)

**注意：**

*flannel下载成功以后，master状态还是NotReady则说明还没就绪，需要等待一会儿，然后节点就处于就绪状态了。*

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/e7e29c6f-727f-466e-8b35-10fd406a0c73)

### 6.5 将Node节点加入集群
分别在k8s-node01和k8s-node02主机上执行kubeadm init最后输出的kubeadm join命令，将node节点加入到Kubernetes master集群中。
```
kubeadm join 192.168.1.110:6443 --token rawp9y.0223try0okckj9kx \
        --discovery-token-ca-cert-hash sha256:de7f72ae8edd7bdbaa66f3bd168a80cee4f3aaf3c2c929ece55160265e9ca7ea
```

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/3fce5134-af1d-47a0-bd45-7d7557b7f616)

**注意：**
*默认的token有效期为24小时。当过期之后，该token就不能用了，这时可以使用如下命令创建token.*
```
kubeadm token create --print-join-command
```
*创建一个永不过期的token*
```
kubeadm token create --ttl 0
```
### 6.6 验证集群
使用kubectl get nodes命令查看节点状态。

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/73fd5a5c-a95d-43bd-adf5-4a52c32d92c2)

查看所有命名空间的所有pod:
```
kubectl get pods --all-namespaces -o wide
```

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/c8891da2-ac3a-4cf4-b028-d16d42839d3d)

 查看集群健康状态:
```
kubectl get cs
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/659f308e-4c03-470b-9c28-0a3124da4438)

## 7 安装Dashboard（master节点）
Dashboard是k8s可视化工具，用来查看集群信息。所以，Dashboard不是必需组件。github仓库地址为：
```
https://github.com/kubernetes/dashboard
```
### 7.1 下载recommended.yaml文件
```
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.4/aio/deploy/recommended.yaml
```
### 7.2 新增集群外部访问接口
默认Dashboard只能集群内部访问，为了能够从集群外部也能访问Dashboard，修改其中kubernetes-dashboard的service，指定nodePort端口为30218，新增type类型为nodePort。
```
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort //新增
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30218 //新增
  selector:
    k8s-app: kubernetes-dashboard
```
配置项说明：

|       参数                   |   说明                                                         |   
|------------------------------|----------------------------------------------------------------|
|  apiVersion                  |  指定api版本，此值必须在kubectl api-versions中                   |  
|  kind                        |  指定创建资源的角色/类型                                         |  
|  metadata                    |  资源的元数据/属性                                               |  
|  name                        |  资源Service类型的名称，在同一个Namespace中必须唯一                | 
|  namespace                   |  资源Service所属的命名空间                                       | 
|  spec                        |  资源规范字段                                                    | 
|  selector                    |  标签选择器，用于确定当前Service代理哪些Pod，仅适用于ClusterIP、NodePort和LoadBalancer类型。如果类型为ExternalName，则忽略。 | 
|  ports                       |  Service对外暴露的端口列表。 | 
|  port                        |  Service服务器监听的端口，Service在集群内部暴露的端口。 | 
|  targetPort	                 |  Pod端口容器暴露的端口。 | 
|  nodePort	                   |  对外暴露的端口好，如果不指定端口号则随机生成一个端口号。 | 
|  type  	                     |  NodePort类型可以对外暴露，让外部访问这个集群，主要有ClusterIP、NodePort、LoadBalancer和ExternalName，默认ClusterIP。 | 

### 7.3 部署Dashboard
```
kubectl apply -f recommended.yaml
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/96ff608d-7f1b-45b7-b4c4-babcc4403245)

查看pod,svc状态:
```
kubectl get pod,svc -n kubernetes-dashboard
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/33002d76-2495-4adf-8a0d-2be897e415fd)

如果配置有问题则需要删除Dashboard
```
# 查询 Pod 
kubectl get pods --all-namespaces | grep "dashboard"

# 删除 Pod

kubectl delete deployment kubernetes-dashboard  --namespace=kubernetes-dashboard
kubectl delete deployment dashboard-metrics-scraper --namespace=kubernetes-dashboard

# 查询 service

kubectl get service -A

# 删除 service
kubectl delete service kubernetes-dashboard  --namespace=kubernetes-dashboard
kubectl delete service dashboard-metrics-scraper  --namespace=kubernetes-dashboard

# 删除账户和密钥
kubectl delete sa kubernetes-dashboard --namespace=kubernetes-dashboard
kubectl delete secret kubernetes-dashboard-certs --namespace=kubernetes-dashboard
kubectl delete secret kubernetes-dashboard-key-holder --namespace=kubernetes-dashboard
```
访问指定的30218端口，master节点IP为192.168.1.100,可以通过Chrome浏览器访问：
```
https://192.168.1.110:30218
```
### 7.4 解决证书问题(自签证书)
创建http.ext文件,需要指定要访问的IP：
```
cat >./http.ext << EOF
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName=@SubjectAlternativeName

[SubjectAlternativeName]
IP.1=127.0.0.1
IP.2=192.168.1.110
EOF
```
生成自签证书
```
openssl req -new -newkey rsa:2048 -sha256 -nodes -out dashboard.csr -keyout dashboard.key -subj "/C=CN/ST=Shanghai/L=Shanghai/O=B-Soft Inc./OU=Web Security/CN=192.168.1.110"
openssl x509 -req -days 3650 -in dashboard.csr -signkey dashboard.key -out dashboard.crt -extfile http.ext
openssl pkcs12 -export -in dashboard.crt -inkey dashboard.key -out dashboard.p12 -name bsoft-media
```
其中客户端浏览器需要安装证书dashboard.p12。
查看原有证书。
```
kubectl get secret kubernetes-dashboard-certs -n kubernetes-dashboard
```

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/b421e651-30e7-4628-a5d9-2fbacf64bd14)

 删除原有证书
```
kubectl delete secret kubernetes-dashboard-certs -n kubernetes-dashboard
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/bb9f7fa5-48ec-41f1-9e19-a37ceda1bbba)

通过新生成的证书创建secret
```
kubectl create secret generic kubernetes-dashboard-certs --from-file=dashboard.key --from-file=dashboard.crt -n kubernetes-dashboard
```

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/d375baa9-cd42-4d79-9405-e8cec3b22f6b)

查看dashboard的pod
```
kubectl get pod -n kubernetes-dashboard  | grep dashboard
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/35a9cc8a-96c9-4fef-aab7-ef4c417a8b05)

删除原有的pod（删除后会自动创建新的pod）
```
kubectl delete pod kubernetes-dashboard-5fc4c598cf-4z7n2 -n kubernetes-dashboard
```

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/98c1c38a-e8c5-4ec5-8d9a-49a4a59a6f5c)

 再次查看，pod 正在创建中
```
kubectl get pod -n kubernetes-dashboard  | grep dashboard
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/6b603897-b4e5-422c-8003-76638ca00cc1)

再次查看pod创建好了的状态

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/f3c2ce07-83b7-4779-bb13-210a9d79b745)

 重新访问地址：
```
https://192.168.1.110:30218
```
Kubernetes仪表盘成功打开

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/f55f6c42-bd77-4379-8095-530f94a1e921)

### 7.5 登录Dashboard
创建账号
```
kubectl create serviceaccount dashboard-admin -n kubernetes-dashboard
```

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/5d6830c9-1870-4cd7-918a-40f22160b6ba)

用户授权
```
kubectl create clusterrolebinding dashboard-admin-rb --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:dashboard-admin
```

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/b7792489-82df-41d3-942a-ca05f0f6c750)

获取登录Dashboard登录token
```
kubectl describe secrets -n kubernetes-dashboard $(kubectl -n kubernetes-dashboard get secret | awk '/dashboard-admin/{print $1}')
```
输出如下
```
Name:         dashboard-admin-token-t8gqt
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: dashboard-admin
              kubernetes.io/service-account.uid: b32f1414-0e2e-4351-9ca5-2dd8f5b5d52f

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1099 bytes
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IjNycjI4ZDJOYlVvUWpacUtFbndUOC1BUmFlV3hCMGV1WEg3UERFb1hBLTAifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tdDhncXQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiYjMyZjE0MTQtMGUyZS00MzUxLTljYTUtMmRkOGY1YjVkNTJmIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmVybmV0ZXMtZGFzaGJvYXJkOmRhc2hib2FyZC1hZG1pbiJ9.bAD6UVLZCOCFl1nkUAEtYvFQJ6rjwx1f9aXJVvNSQSAcQvIg9QN9hh0fP36DjU-A_3rCQ7LenRTbGlmmLKB2GFWE3Xvt8HMc2zlmWPZPIVoOEZsF6_BrrvY-OR5SFMyS45JUIA5-EBfSD3goi_RTv3VJI0sC6eecN12lnhy8paOn2xDqClIjiN7enFlAz-IdhBFGIC_oA28odEN3u8WtVR9L8bNuAhSw8X_UaBKGDEkSYWeL3qkWPvN_Hn8_du_cQFVbj5Y1dBsDdq_RJ8Q1h-7FH0KM-ErScFR3SF7ZC4n9DxO21BGReM2ZhWzZaYEVdQburoR2SVhUcqFEFRAX4A
```
使用token登录Dashboard

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/0af63d5a-a2d9-4899-8be3-629fb4cf3a43)

## 8 使用Deployment部署nginx服务
快速部署一个应用大致流程为：制作镜像(Dockerfile)-->使用控制器部署镜像(Deployment)-->对外暴露应用(Service)-->日常运维。即，使用Dockerfile构建镜像，然后可以上传到镜像仓库，再使用k8s控制器（Deployment）去部署镜像，然后再使用Service去对外暴露应用，最后进行应用监控、收集日志等。

- 在master节点上使用Deployment控制器部署镜像
```
kubectl create deployment nginx --image=nginx --replicas=3
#参数说明：
#    deployment后面的nginx表示Deployment名称，可以是任何名字。
#    image参数指定的是镜像名称。
#    reblicas参数表示副本数，可以理解为需要创建几个容器。
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/6e850a92-bc75-489b-a51d-ea25c8ddfdcd)

- 查看pod
```
kubectl get pods
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/9a947436-4570-4972-b57d-0b43b531d90b)

 前面create的时候指定了3个副本，所以这里创建了3个容器。如果Pod的STATUS状态为ContainerCreating 则表示容器还处在启动状态。请求启动完毕后STATUS状态为Running:

 ![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/5b42f695-733f-4733-a8d2-a8c0665c5cf6)

- 查看Pod详细情况

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/6b8ac88a-2f49-48dd-8f97-bcaaf5e19ac6)

- 在node节点中查看镜像是否成功（下图是成功拉取到，并成功启动容器）。

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/89d64826-0b45-4a24-9fa8-5c3f5f845207)

- 查看replicas状态
```
kubectl get replicaset
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/3067a3ba-cd87-4c64-acb0-40ef7e9c0326)

- 使用Service暴露Pod
```
# --port 这个端口是 K8s 集群内部使用的，这里我们还用不到，但是必须指定的。
# --target-port是镜像中服务运行的端口，比如 nginx 默认端口是 80，这里就写 80 端口。
# --type=NodePort 是通过 NodePort 类型，在 Pod 所在的节点进行绑定端口，让外部客户端访问 Pod
kubectl expose deployment nginx --port=80 --target-port=80  --type=NodePort
```
- 查看Service
```
kubectl get service
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/e7544eac-3e49-4da7-be0a-b9b83231fe17)

图中nginx服务PORTS部分解释：80端口用于node之间通信，比如当前有多个node节点，node之间对该nginx应用进行访问的时候使用80端口。而后面的30042这个端口用于外部对nginx的访问，例如通过浏览器访问时，80端口是不能访问的，此时必须通过30042这个端口访问，而这个端口也是随机生成的。'

- 访问nginx
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/f2c4cab8-8c68-46f7-a336-d96b318fbac3)

 - 删除nginx服务

集群中不需要某个服务时，可以将服务删除
```
# 删除 service
kubectl delete service nginx
# 删除 nginx 的控制器
kubectl delete deployment nginx
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/f7411513-69a6-4b29-b273-6b808e3244d4)

node节点上nginx容器也已删除

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/6e8790fe-44f3-48c2-83d9-4fa4c8c21422)

## 9 使用Deployment升级nginx版本
当集群中某个服务要升级时，则需要停止目前与该服务相关的所有Pod，然后下载新版本镜像并创建新的Pod。集群规模比较大，需要先全部停止然后逐步升级的方式会导致较长时间的服务不可用。K8s提供了滚动升级功能来解决上述问题。

滚动升级是K8s对Pod升级的默认策略，通过使用新版本Pod逐步更新旧版本Pod，实现零停机发布，对于用户来说也是无感知的。

### 9.1 部署nginx
- 创建Deployment时，自动创建ReplicaSet，Pod由ReplicaSet创建和管理。
```
cat > nginx-deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.16
        name: nginx

---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 31100
  type: NodePort
EOF
```
- 部署nginx
```
kubectl apply -f nginx-deployment.yaml
```
- 查看当前运行的Pod和Service，输入下图信息时表示已经部署成功

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/f8c45df9-aa9d-45cd-ad3e-757655ab74f5)

- 在node节点上查看镜像和容器状态

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/223fce53-c815-4028-b92e-3b081c92d8c1)

- 查看Pod详情，可以看到nginx是指定的版本
```
kubectl describe pod web-78449c65d4-kg7rc | tail
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/e8d146f9-ee1f-4523-be08-28b10db207d5)

- 浏览器访问nginx
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/13a322c6-0e3e-4c38-98c7-0b3b4b9ac60a)

### 9.2 通过配置文件方式滚动升级nginx
- 修改配置文件nginx-deployment.yaml，把Pod镜像更新为1.17

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/7b5b2c41-293d-45b8-b4c6-55c9c2aec5f1)

- 更新nginx版本
```
kubectl apply -f nginx-deployment.yaml
```
- 验证 

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/8f808d7c-31a8-4540-8cd3-5f3a4cbe8c1e)

- 查看节点镜像

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/24b24311-3e76-405d-849b-551eb2b078dd)

- 查看nginx更新过程

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/51c39309-5cea-46d7-b975-645be9efa87e)

 Deployment升级Pod过程：
 
1. 初始化创建Deployment时，创建了一个ReplicaSet，ReplicaSet管理Pod，所以ReplicaSet帮我们建立了3个Pod副本。
2. 当更新Deployment时创建新的ReplicaSet，将心的副本数设置为1，然后将旧的ReplicaSet副本数设置为2。
3. 后续的步骤安装相同的策略对新旧两个ReplicaSet进行逐步调整。
4. 最后新的ReplicaSet运行了3个新的Pod副本，旧的ReplicaSet副本数则缩减为0。

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/114573cd-0945-44d8-9b0d-cd88ecf50b79)

 整个升级过程由Deployment控制器完成。主要流程即：创建新ReplicaSet然后扩容，同时之前的ReplicaSet会缩减到0.旧的ReplicaSet仍会保留，供回滚时使用。

 ### 9.3 通过命令方式滚动升级nginx
 - 升级至1.18
```
# kubectl set image 资源类型+名字（web）原镜像名=新镜像名:版本号
kubectl set image deployment web nginx=nginx:1.18
```
- 验证

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/6580c9c2-5632-4b40-84e2-c1b6eed26c34)

## 10 使用Deployment回滚nginx版本
当我们更新nginx至最新版本时，发现最新版本不稳定。这时就需要回滚至上一个版本。
- 查看nginx历史版本
```
kubectl  rollout  history  deployment web 
```

  ![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/eac804b6-31a7-46b7-8cce-43394c8f6c75)

- 回滚历史版本
```
kubectl  rollout undo  deployment  web 
```
- 验证（回滚到1.17版本）

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/2c36717c-4ce6-43d4-9243-0ad8d010d1bb)

再次查看nginx历史版本是，发现版本2被删除了，新增了历史版本4，即现在回滚成功，该Department回滚到第2个版本。

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/7d4b473c-f2c9-4909-818a-71bcfa90a52f)

- 回滚到指定版本
```
kubectl rollout undo deployment/web --to-revision=3 
```
- 验证（回滚到1.18版本）

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/5e92f940-a940-44f4-9039-dc737ab1664f)

- 处理更新版本后CHANGE-CAUSE为none的问题

```
kubectl set image deployment web nginx=nginx:1.19 --record=true
```

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/88261c02-8c78-4107-9348-a7ad5149f352)

这样就记录了我们做了哪些操作。进行回滚的时候可以知道该回滚到哪个版本，比none要清楚一点。

## 11 使用Deployment水平扩容和缩容
