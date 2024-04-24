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
## 2 部署步骤
### 2.1 创建Namespace
- 编写01-mysql-namespace.yaml文件
```
cat >> 01-mysql-namespace.yaml << EOF
apiVersion: v1
#创建Namespace类型资源
kind: Namespace
metadata:
  #资源名称
  name: mysql
  #标签为app:mysql
  labels:
    app: mysql
EOF
```
- 创建Namespace
  ```
  kubectl apply -f 01-mysql-namespace.yaml
  ```
- 查看Namespace
```
kubectl get ns
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/1b86ead1-7853-48fb-a343-3f7a877b5e6d)

### 2.2 配置ConfigMap
- 编写02-mysql-configmap.yaml文件
```
cat >> 02-mysql-configmap.yaml << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql
  namespace: mysql
  labels:
    app: mysql
data:
  #这里定义了多个数据信息
  master.cnf: |
    # Master配置
    [mysqld]
    datadir=/var/lib/mysql
    pid-file=/var/run/mysqld/mysqld.pid
    socket=/var/run/mysql/mysql.sock
    log-error=/var/log/mysql/error.log
    log-bin=mysqllog
    skip-name-resolve
    lower-case-table-names=1
    log_bin_trust_function_creators=1
  slave.cnf: |
    # Slave配置
    [mysqld]
    datadir=/var/lib/mysql
    pid-file=/var/run/mysqld/mysqld.pid
    socket=/var/run/mysql/mysql.sock
    log-error=/var/log/mysql/error.log
    super-read-only
    skip-name-resolve
    log-bin=mysql-bin
    lower-case-table-names=1
    log_bin_trust_function_creators=1
EOF
```
- 创建MySQL配置
```
kubectl apply -f 02-mysql-configmap.yaml
```
- 查看mysql命名空间下的ConfigMap
```
kubectl get cm -n mysql
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/7e160d00-e925-4059-9aba-4202c49caecc)

- 查看mysql命名空间下名为mysql的configmap详情
```
kubectl describe configmap mysql -n mysql
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/123aadc2-f852-48c7-9f49-cf1e714481da)

### 2.3 创建MySQL用户密码
- 编写secret文件：03-mysql-secret.yaml
```
cat >> 03-mysql-secret.yaml << EOF
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: mysql
  labels:
    app: mysql
#Opaque 类型的数据是一个 map 类型，要求value是base64编码。
type: Opaque
data:
  password: YnNvZnRAMTEw # bsoft@110 转成base64: echo -n "bsoft@110" | base64
  #主从用的账号
  replicationUser: Y29weQ== #copy
  replicationPassword: YnNvZnRAMTEw # bsoft@110
EOF
```
文件中创建了两个账号及其相对应的密码，后面用来登录主库和从库
1. root/bsoft@110
2. copy/bsoft@110
