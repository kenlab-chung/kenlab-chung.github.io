# K8s部署MySQL集群
环境说明：

操作系统：CentOS 7.9

K8s集群：参考[K8s集群部署指南](./K8sClusterDeployment.md)
## 1安装虚拟机
- Deiban 12 安装虚拟机
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
 
- Debian 11 安装虚拟机
```
sudo apt install curl wget gnupg2 lsb-release -y
curl -fsSL https://www.virtualbox.org/download/oracle_vbox_2016.asc|sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/vbox.gpg
curl -fsSL https://www.virtualbox.org/download/oracle_vbox.asc|sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/oracle_vbox.gpg

echo "deb [arch=amd64] http://download.virtualbox.org/virtualbox/debian $(lsb_release -cs) contrib" | sudo tee /etc/apt/sources.list.d/virtualbox.list

sudo apt update
sudo apt install linux-headers-$(uname -r) dkms -y
sudo apt install virtualbox-6.1 -y
```

## 2 部署远程桌面
Debian12部署tigerVNC Server远程虚拟桌面。方便windows远程操作。

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

- 远程桌面效果如图
  
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/b155ba83-79f7-4f9b-9e11-3713e59231ac)
 
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
```
kubectl get pv,pvc
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/3d5155a4-8cc3-495f-87ab-4ce5c3da1b04)

### 3.4 创建Deployment和Service
```
cat >>  mysql-deploy.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        env:           #配置变量，设置mysql的密码
        - name: MYSQL_ROOT_PASSWORD
          value: bsoft-mysql
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql     #MySQL容器的数据都是存在这个目录的，要对这个目录做数据持久化
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysqlpvc           #指定pvc名称
---
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  selector:
    app: mysql
  type: NodePort
  ports:
  - port: 3306
    targetPort: 63306
EOF
kubectl apply -f mysql-deploy.yaml
```
查看pod状态

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/79237272-ad1a-4c89-9b07-14b7744193af)

### 3.5 进入数据库，添加数据
```
kubectl exec -it mysql-794754cdd-d6wrd -- /bin/bash
mysql -u root -pbsoft-mysql
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/0530e522-160c-4254-b8e5-75c2cae58cc9)
### 3.6 手动删除节点，验证数据持久化功能
在node02上的删除容器，由于deployment保证副本数量。所以会重新调度到node02上。
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/f88073df-ce8c-46c7-a2de-342f50d9f7f6)

登录数据库，查看数据是否存在。

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/37307739-9cc3-46b5-8084-1307373147ef)

上图表明，数据库实例恢复后，原来创建的数据和数据库依然完好。

