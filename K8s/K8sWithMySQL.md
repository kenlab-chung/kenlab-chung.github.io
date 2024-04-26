# K8s部署MySQL集群
## 1 MySQL数据持久化
### 1.1 搭建nfs实现mysql数据持久化
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
