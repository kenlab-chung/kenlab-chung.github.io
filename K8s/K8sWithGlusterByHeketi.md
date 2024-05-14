#  通过Heketi管理GlusterFS为K6s集群提供持久化存储
## 1 Heketi简介
Heketi是一个提供RESTful API管理GlusterFS卷的框架，便于管理员对GlusterFS进行操作：

- 可以用于管理GlusterFS卷的生命周期;
- 能够在OpenStack，Kubernetes，Openshift等云平台上实现动态存储资源供应（动态在GlusterFS集群内选择bricks构建volume）；
- 支持GlusterFS多集群管理。

 框架：

 ![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/3fea9dbc-3d7d-4d0d-9971-a0e2e029dd5a)

- Heketi支持GlusterFS多集群管理；
- 在集群中通过zone区分故障域。

## 2 环境

**注意：Glusterfs只需要安装并启动即可，不必组建存储池。**
集群IP地址规划（所有主机关闭防火墙和Selinux）
|       IP       |   操作系统  |   主机名称  | 
|----------------|-------------|-------------|
|  192.168.1.28  |  CentOS7.9  |  k8s-master |
|  192.168.1.42  |  CentOS7.9  |  k8s-node01 | 
|  192.168.1.34  |  CentOS7.9  |  k8s-node02 | 
|  192.168.1.113 |  CentOS7.9  |    node1    | 
|  192.168.1.48  |  CentOS7.9  |    node2    | 
|  192.168.1.49  |  CentOS7.9  |    node3    | 
|  192.168.1.50  |  CentOS7.9  |    node4    | 
|  192.168.1.113 |  CentOS7.9  |    heketi   | 

## 3 部署Heketi(192.168.1.113 节点)
### 3.1 安装heketi
```
#CentOS默认无heketi源，添加源及安装
yum install -y centos-release-gluster
yum install -y heketi heketi-client
```

