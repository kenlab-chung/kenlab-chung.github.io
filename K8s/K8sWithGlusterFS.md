# K8s之GlusterFS集群部署
## 1 部署GLusterFS集群
### 1.1 服务器节点分配
|  服务器节点 |   操作系统  | 主机名 |               磁盘部署                  |                     对应挂载点              |    IP地址   | 
|-------------|-------------|--------|----------------------------------------|---------------------------------------------|--------------|
|  Node1节点  |  CentOS7.9  |  node1 | /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1 | /data/sdb1 /data/sdc1 /data/sdd1 /data/sde1 |192.168.1.113|
|  Node2节点  |  CentOS7.9  |  node2 | /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1 | /data/sdb1 /data/sdc1 /data/sdd1 /data/sde1 |192.168.1.48 |
|  Node3节点  |  CentOS7.9  |  node3 | /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1 | /data/sdb1 /data/sdc1 /data/sdd1 /data/sde1 |192.168.1.49|
|  Node4节点  |  CentOS7.9  |  node4 | /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1 | /data/sdb1 /data/sdc1 /data/sdd1 /data/sde1 |192.168.1.50|
### 1.2 服务器环境(所有节点)
- 关闭防火墙
```
systemctl stop firewalld
setenforce 0
```
- 添加磁盘

给每个node添加磁盘后，重启系统。
```
lsblk
```
可以看到系统已经识别到新增加的磁盘。

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/b26895c5-7e06-49d5-807b-60a87a4b961d)

- 磁盘分区和挂载
```
vim fdisk.sh
``` 
```
#!/bin/bash
NEWDEV=`ls /dev/sd* | grep -o 'sd[b-z]' | uniq`
for VAR in $NEWDEV
do
   echo -e "n\np\n\n\n\nw\n" | fdisk /dev/$VAR &> /dev/null
   mkfs.xfs /dev/${VAR}"1" &> /dev/null
   mkdir -p /data/${VAR}"1" &> /dev/null
   echo "/dev/${VAR}"1" /data/${VAR}"1" xfs defaults 0 0" >> /etc/fstab
done
mount -a &> /dev/null
```
chmod a+x ./fdisk.sh
```
chmod a+x ./fdisk.shdf
./fdisk.sh
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/d579c050-c63b-494d-9eb3-bd9c0077101c)

- 修改主机名，配置hosts文件
```
# 设置 192.168.1.113 主机名
hostnamectl set-hostname  node1
# 设置 192.168.1.48 主机名
hostnamectl set-hostname  node2
# 设置 192.168.1.49 主机名
hostnamectl set-hostname  node3
# 设置 192.168.1.50 主机名
hostnamectl set-hostname  node4
```
```
cat >> /etc/hosts << EOF
192.168.1.113 node1
192.168.1.48 node2
192.168.1.49 node3
192.168.1.50 node4
EOF
```
### 1.3 部署GlusterFS（所有节点）
```
yum install -y centos-release-gluster
yum install -y glusterfs-server
systemctl start glusterd
systemctl enable glusterd
systemctl status glusterd
```
### 1.4 添加节点到存储信任池(在node1节点上操作)
只要在一台node节点上添加其它节点即可
```
gluster peer probe node2
gluster peer probe node3
gluster peer probe node4
```
分别在四个节点上查询节点状态
```
gluster peer status
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/01e33e66-ab5c-470d-bf68-1f808b8d9c47)

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/341b1e5a-3efd-452e-b012-69b3e0f108a5)

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/5cf95b38-f58f-44f2-bcbc-c76baf065bd9)

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/47392d78-74b9-40ca-aa12-44f8c1108c45)



