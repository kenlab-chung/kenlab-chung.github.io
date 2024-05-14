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
- 分发公钥至GlusterFS主机 -i ：指定公钥
```
ssh-copy-id -i /etc/heketi/heketi_key.pub root@192.168.1.113
ssh-copy-id -i /etc/heketi/heketi_key.pub root@192.168.1.48
ssh-copy-id -i /etc/heketi/heketi_key.pub root@192.168.1.49
ssh-copy-id -i /etc/heketi/heketi_key.pub root@192.168.1.50
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
## 4 设置GlusterFS集群
### 4.1 创建topology.json文件
通过topology.json文件定义组建GlusterFS集群；

topology指定了层级关系：clusters-->nodes-->node/devices-->hostnames/zone；

node/hostnames字段的manage填写主机ip，指管理通道，在heketi服务器不能通过hostname访问GlusterFS节点时不能填写hostname；

node/hostnames字段的storage填写主机ip，指存储数据通道，与manage可以不一样；

node/zone字段指定了node所处的故障域，heketi通过跨故障域创建副本，提高数据高可用性质，如可以通过rack的不同区分zone值，创建跨机架的故障域；

devices字段指定GlusterFS各节点的盘符（可以是多块盘），必须是未创建文件系统的裸设备；

本次四台glusterfs主机添加硬盘`/dev/sdf`和`/dev/sdg`

```
#cat /etc/heketi/topology.json
{
	"clusters": [
		{
			"nodes": [
				{
					"node": {
						"hostnames": {
							"manage": [
								"192.168.1.113"
							],
							"storage": [
								"192.168.1.113"
							]
						},
						"zone": 1
					},
					"devices": [
						"/dev/sdf",
						"/dev/sdg"
					]
				},
				{
					"node": {
						"hostnames": {
							"manage": [
								"192.168.1.48"
							],
							"storage": [
								"192.168.1.48"
							]
						},
						"zone": 1
					},
					"devices": [
						"/dev/sdf",
						"/dev/sdg"
					]
				},
				{
					"node": {
						"hostnames": {
							"manage": [
								"192.168.1.49"
							],
							"storage": [
								"192.168.1.49"
							]
						},
						"zone": 1
					},
					"devices": [
						"/dev/sdf",
						"/dev/sdg"
					]
				},
				{
					"node": {
						"hostnames": {
							"manage": [
								"192.168.1.50"
							],
							"storage": [
								"192.168.1.50"
							]
						},
						"zone": 1
					},
					"devices": [
						"/dev/sdf",
						"/dev/sdg"
					]
				}
			]
		}
	]
}
```
导入配置文件，使配置生效。
```
heketi-cli --server http://localhost:8080 --user admin --secret admin@123 topology load --json=/etc/heketi/topology.json
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/d6f23447-101f-4a3c-bd89-c1a0c97b9af4)
