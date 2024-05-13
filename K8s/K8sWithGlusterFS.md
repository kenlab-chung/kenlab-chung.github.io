# K8s之GlusterFS集群部署
## 1 部署GLusterFS集群
### 1.1 服务器节点分配
glusterFS版本： 9.6
|  服务器节点 |   操作系统  | 主机名 |               磁盘部署                  |                     对应挂载点              |    IP地址   | 
|-------------|-------------|--------|----------------------------------------|---------------------------------------------|--------------|
|  Node1节点  |  CentOS7.9  |  node1 | /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1 | /data/sdb1 /data/sdc1 /data/sdd1 /data/sde1 |192.168.1.113|
|  Node2节点  |  CentOS7.9  |  node2 | /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1 | /data/sdb1 /data/sdc1 /data/sdd1 /data/sde1 |192.168.1.48 |
|  Node3节点  |  CentOS7.9  |  node3 | /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1 | /data/sdb1 /data/sdc1 /data/sdd1 /data/sde1 |192.168.1.49 |
|  Node4节点  |  CentOS7.9  |  node4 | /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1 | /data/sdb1 /data/sdc1 /data/sdd1 /data/sde1 |192.168.1.50 |
|  客户端节点  |  CentOS7.9  | client |                                         |                                             |192.168.1.33 |
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
### 1.4 添加节点到存储信任池(node1节点)
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

### 1.5 创建卷（node1节点）
根据规划创建如下卷：
|      卷名称    |    卷类型   | Brick                                                                       | 
|----------------|-------------|-----------------------------------------------------------------------------|
|  dis-volume    |  分布式卷    | node1(/data/sdb1)、node2(/data/sdb1)                                        | 
|  stripe-volume |  条带卷      | node1(/data/sdc1)、node2(/data/sdc1)                                        | 
|  rep-volume    |  复制卷      |  node3(/data/sdb1)、node4(/data/sdb1)                                       |
|  dis-stripe    |  分布式条带卷 |  node1(/data/sdd1)、node2(/data/sdd1)、node3(/data/sdd1)、node4(/data/sdd1) | 
|  dis-rep       |  分布式复制卷 |  node1(/data/sde1)、node2(/data/sde1)、node3(/data/sde1)、node4(/data/sde1) | 
#### 1.5.1 创建分布式卷
无需指定类型，默认创建的是分布式卷
```
#创建分布式卷
gluster volume create dis-volume node1:/data/sdb1 node2:/data/sdb1 force
#查看卷列表
gluster volume list
#启动卷
gluster volume start dis-volume
#查看卷信息
gluster volume info dis-volume
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/b6900589-7a5e-47b0-8514-0f2df1ab3700)

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/5a2f534e-09fb-45a6-bea8-5e8b99fa9381)

#### 1.5.2 创建条带卷
指定类型为stripe，数值为2，且后面跟了2个Brick Server，所以创建的是条带卷
```
gluster volume create stripe-volume stripe 2 node1:/data/sdc1 node2:/data/sdc1 force
```
已不支持

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/ae504f0a-b691-4094-987a-8ef2bf147852)

#### 1.5.3 创建复制卷
指定类型为replica，数值为2，且后面跟了2个Brick Server，所以创建的是复制卷
```
gluster volume create rep-volume replica 2 node3:/data/sdb1 node4:/data/sdb1 force
gluster volume list
gluster volume start rep-volume
gluster volume info rep-volume 
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/b5d8e8b2-8cd1-4920-946a-72e018e81dc1)

#### 1.5.4 创建分布式条带卷
指定类型为stripe，数值为2，而且后面跟了4个Brick Server，是2的两倍，所以创建的是分布式条带卷
```
gluster volume create dis-stripe stripe 2 node1:/data/sdd1 node2:/data/sdd1 node3:/data/sdd1 node4:/data/sdd1 force
```
已不支持

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/cc9c4fa8-6d9b-4893-9aa7-ca861513615e)
#### 1.5.5 创建分布式复制卷
指定类型为replica，数值为2，而且后面跟了4个Brick Server，是2的两倍，所以创建的是分布式复制卷
```
gluster volume create dis-rep replica 2 node1:/data/sde1 node2:/data/sde1 node3:/data/sde1 node4:/data/sde1 force
gluster volume list
gluster volume start dis-rep
gluster volume info dis-rep
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/4e9b0723-46d8-42a0-a82e-06798a0351ac)
## 2 部署GLuster客户端
### 2.1 安装客户端软件
```
yum install -y centos-release-gluster
yum install -y glusterfs glusterfs-server
systemctl start glusterd
systemctl enable glusterd
systemctl status glusterd
```
### 2.2 创建挂载目录
```
mkdir -p /test/{dis,stripe,rep,dis_stripe,dis_rep}
```
### 2.3 配置hosts文件
```
cat >> /etc/hosts << EOF
192.168.1.113 node1
192.168.1.48 node2
192.168.1.49 node3
192.168.1.50 node4
EOF
```
### 2.4 挂载Gluster文件系统
#### 2.4.1 临时挂载
```
mount.glusterfs node1:dis-volume /test/dis
mount.glusterfs node1:rep-volume /test/rep
mount.glusterfs node1:dis-rep /test/dis_rep
df -hT
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/8750d0bf-a801-4dbf-97af-b09fc457f4ea)

#### 2.4.2 永久挂载
在`/etc/fstab`末行写入
```
node1:dis-volume		/test/dis				glusterfs		defaults,_netdev		0 0
node1:rep-volume		/test/rep				glusterfs		defaults,_netdev		0 0
node1:dis-rep			/test/dis_rep			glusterfs		defaults,_netdev		0 0
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/92d8cb72-0679-4b7e-99cb-29bcb97992cb)
## 3 测试GLuster文件系统
### 3.1 在卷中写入文件系统(客户端操作)
```
cd /opt
dd if=/dev/zero of=/opt/demo1.log bs=1M count=40
dd if=/dev/zero of=/opt/demo2.log bs=1M count=40
dd if=/dev/zero of=/opt/demo3.log bs=1M count=40
dd if=/dev/zero of=/opt/demo4.log bs=1M count=40
dd if=/dev/zero of=/opt/demo5.log bs=1M count=40

cp /opt/demo* /test/dis/
cp /opt/demo* /test/rep/
cp /opt/demo* /test/dis_rep/
```
### 3.2 查看文件分布
#### 3.2.1 分布式文件分布
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/6b75b304-6ef7-4c4c-a2d3-471ef1fa50be)

