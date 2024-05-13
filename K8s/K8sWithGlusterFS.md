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
df -hT
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/3449bec7-43df-4522-a5dd-5f6e74bc0f1d)
