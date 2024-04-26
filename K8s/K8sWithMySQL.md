# K8s部署MySQL集群
环境说明：
操作系统：CentOS 7.9
K8s集群：参考[K8s集群部署指南](./K8sClusterDeployment.md)
## 1安装虚拟机
为方便测试，本案例在Oracle VirtualBox 7.0 虚拟机上部署K8s集群。 其中实体机服务器安装Debian 12操作系统。
下载虚拟机
```
wget \
https://download.virtualbox.org/virtualbox/7.0.12/virtualbox-7.0_7.0.12-159484~Debian~bookworm_amd64.deb \
-P ~/Downloads/
```
更新系统
```
sudo apt update
```
安装虚拟机
```
sudo apt install ./virtualbox-7.0_7.0.12-159484~Debian~bookworm_amd64.deb
```
遇到下图故障时，执行下列指令，并重启服务器
```
sudo /sbin/vboxconfig
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/81d7f313-1ccf-4423-bbb2-562253a7dea1)
 

## 2 部署远程桌面
Debian12部署igerVNC Server远程虚拟桌面。方便windows远程操作。

- 安装环境和vnc server工具
```
sudo apt install gnome gdm3
sudo apt install tigervnc-standalone-server
```
- 设置登录密码
```
vncserver 
```
- 配置
```
#启动语句
vncserver -geometry 1280x1024 -localhost no :2
#关闭语句
vncserver -kill :2
```
- 使用RealVNC Viewer 远程Debian 12 服务器
```
https://www.realvnc.com/en/connect/download/viewer/
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/1556c929-6b9d-4c31-a6da-ae3c74b95105)
  
## 3 MySQL数据持久化
### 3.1 搭建nfs实现mysql数据持久化
nfs服务器端一般部署在master节点
```
 在master执行
 yum -y install nfs-utils rpcbind
 mkdir /nfsdata/mysql -p
 echo /nfsdata *(rw,no_root_squash,sync) > /etc/exports  
 systemctl restart nfs-server rpcbind 
 systemctl enable rpcbind && systemctl enable nfs-server
 showmount -e
 Export list for master:
 /nfsdata *

```
在所有node节点部署nfs客户端
```
yum -y install nfs-utils rpcbind 
systemctl restart nfs-server  rpcbind 
```
### 3.2 创建PV
server: IP为master的IP
```
cat >> mysql-pv.yaml << EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysqlpv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle   #Retain 手动删除
  storageClassName: nfs
  nfs:
    path: /nfsdata/mysql
    server: 192.168.1.100
EOF
#kubectl apply -f mysql-pv.yaml
```
查看创建的pv,这是状态处于于available状态
```
kubectl get pv
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/adbe549f-009e-44c6-90dd-b1f3b1663467)

### 3.3 创建PVC
```
cat >> mysql-pvc.yaml << EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysqlpvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 4Gi
  storageClassName: nfs
EOF
#kubectl apply -f mysql-pvc.yaml
```
查看pv,pvc状态，此时pv处于bound状态
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/3d5155a4-8cc3-495f-87ab-4ce5c3da1b04)
