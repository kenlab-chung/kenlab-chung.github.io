# K8s部署MySQL主从结构
## 1 需求与方案
### 1.1 需求
- 启动顺序有要求，master节点必须比slave节点先启动 （statefulset）
- 节点宕机，新的pod启动必须使用原先pod的资源（持久化保证数据不丢失）
- master与slave配置不一样
- master启动之后需要设置主从授权账户，slave需要执行change master命令，以及加入主从的命令
- 希望客户账户名密码自己配置
- slave需要知道master节点的地址
### 1.2 方案
- statefulset：使用statefulSet可以使得pod副本按照编号顺序进行启动，只需要把pod-0作为master即可
- 持久化保证数据不丢失：使用pv和pvc解决，通过pvc与pod的标签进行绑定，一个pod对应一个pvc就可以保证重启后的pod依旧使用原先的资源（测试时使用NFS，生产环境一般使用Ceph和minio）
- 初始化所需的配置信息：使用configmap可以在容器初始化的时候指定需要的配置信息
- 初始化执行的脚步：使用initContainer可以在容器初始化的时候执行需要的脚本
- 密码存放：使用secret可以将密码
- 集群内访问直接podName.serviceName：使用headless service+dns可以让slave节点通过hostname访问master，hostname固定为podName.ServiceName，如：serviceName为mysql，则master的hostname为mysql-0.mysql

