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
### 3.2 配置heketi.json
```
vim /etc/heketi/heketi.json
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/129e3178-b426-4ec4-90bb-04d3866ea307)
### 3.3 设置heketi免密访问GlusterFS
- 选择ssh执行器，heketi服务器需要免密登陆GlusterFS集群的各节点。
```
# -t：秘钥类型；
# -q：安静模式；
# -f：指定生成秘钥的目录与名字，注意与heketi.json的ssh执行器中"keyfile"值一致；
# -N：秘钥密码，””即为空
ssh-keygen -t rsa -q -f /etc/heketi/heketi_key -N ""
```
- heketi服务由heketi用户启动，heketi用户需要有新生成key的读赋权，否则服务无法启动
```
chown heketi:heketi /etc/heketi/heketi_key
```
### 3.4 启动heketi
```
systemctl enable heketi
systemctl start heketi
systemctl status heketi
```
### 3.5 验证
```
curl http://127.0.0.1:8080/hello
```
Hello from Heketi
