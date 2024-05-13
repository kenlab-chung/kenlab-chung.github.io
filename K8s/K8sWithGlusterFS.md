# K8s之GlusterFS集群部署
## 1 部署GLusterFS集群
### 1.1 服务器节点分配
|  服务器节点 |   操作系统  | 主机名 |               磁盘部署                  |                     对应挂载点              |    IP地址   | 
|-------------|-------------|--------|----------------------------------------|---------------------------------------------|--------------|
|  Node1节点  |  CentOS7.9  |  node1 | /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1 | /data/sdb1 /data/sdc1 /data/sdd1 /data/sde1 |192.168.1.113|
|  Node2节点  |  CentOS7.9  |  node2 | /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1 | /data/sdb1 /data/sdc1 /data/sdd1 /data/sde1 |192.168.1.48 |
|  Node3节点  |  CentOS7.9  |  node3 | /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1 | /data/sdb1 /data/sdc1 /data/sdd1 /data/sde1 |192.168.1.49|
|  Node4节点  |  CentOS7.9  |  node4 | /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1 | /data/sdb1 /data/sdc1 /data/sdd1 /data/sde1 |192.168.1.50|
