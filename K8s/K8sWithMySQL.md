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

- 执行命令
```
kubectl apply -f 03-mysql-secret.yaml
```
- 查看mysql命名空间下的secret
  
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/d6ae5168-2052-4992-b6ea-d9f1a2e2a48f)

- 查看mysql命名空间下名为mysql-secret的secret详情
```
kubectl describe secret mysql-secret -n mysql
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/05a90e7c-1af1-473c-a954-a8a8011b801c)

### 2.4 编写initContainer脚本
供在创建StatefulSet中使用。

- 将配置文件拷贝到对应的容器中
```
set -ex
#从pod的hostname中通过正则获取序号，如果没有截取到就退出程序
ordinal=`hostname | awk -F"-" '{print $2}'` || exit 1
#将serverId输入到对应的配置文件中，路径可以随意（与之后的对应上就行），但是文件名不能换
echo [mysqld] > /etc/mysql/conf.d/server-id.cnf
# 由于server-id不能为0，因此给ID加100来避开它
echo server-id=$((100 + $ordinal)) >> /etc/mysql/conf.d/server-id.cnf
if [[ ${ordinal} -eq 0 ]]; then
  # 如果Pod的序号为0，说明它是Master节点，从ConfigMap里把Master的配置文件拷贝到/mnt/conf.d目录下
  cp /mnt/config-map/master.cnf /etc/mysql/conf.d
else
  # 否则，拷贝ConfigMap里的Slave的配置文件
  cp /mnt/config-map/slave.cnf /etc/mysql/conf.d
fi
```
- 初始化mysql集群
```
set -ex
cd /var/lib/mysql
#查看是否存在名为mysqlInitOk的文件，我们自己生产的标识文件，防止重复初始化集群
if [ ! -f mysqlInitOk ]; then
  echo "Waiting for mysqld to be ready（accepting connections）"
  #执行一条mysql的命令，查看mysql是否初始化完毕，如果没有就反复执行直到可以运行
    until mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "use mysql;SELECT 1;"; do sleep 1; done
    echo "Initialize ready"
    #判断是master还是slave
    pod_seq=`hostname | awk -F"-" '{print $2}'`
    if [ $pod_seq -eq 0 ];then
      #创建主从账户
    mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "create user '${MYSQL_REPLICATION_USER}'@'%' identified by '${MYSQL_REPLICATION_PASSWORD}';"
    #设置权限
    mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "grant replication slave on *.* to '${MYSQL_REPLICATION_USER}'@'%' with grant option;"
    #mysql8使用原生密码
    mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "ALTER USER '${MYSQL_REPLICATION_USER}'@'%' IDENTIFIED WITH mysql_native_password BY '${MYSQL_REPLICATION_PASSWORD}';"
    #刷新配置
    mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "flush privileges;"
    #初始化master
    mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "reset master;"
  else
    #设置slave连接的master
    #mysql-0.mysql.mysql的由来{pod-name}.{service-name}.{namespace}
    mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e \
    "change master to master_host='mysql-0.mysql.mysql',master_port=3306, \
    master_user='${MYSQL_REPLICATION_USER}',master_password='${MYSQL_REPLICATION_PASSWORD}', \
    master_log_file='mysql-bin.000001',master_log_pos=156;"
    #重置slave
    mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "reset slave;"
    #开始同步
    mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "start slave;"
    #改成只读模式
    mysql -uroot -p${MYSQL_ROOT_PASSWORD} -e "set global read_only=1;"
  fi
  #运行完毕创建标识文件，防止重复初始化集群
  touch mysqlInitOk
fi
```
### 2.5 创建网络存储服务
采用StorageClass+NFS方式作为网络存储，使用这种方式会自动生成pvc和pv。一般在master节点安装NFS服务端，其它节点安装NFS客户端。
本案例集群信息：
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/944f63ca-2b4f-4d27-885b-4a04f7eadb45)

#### 2.5.1 部署NFS服务（master节点）
- 安装NFS服务
```
# 安装nfs-utils
yum install nfs-utils

# 开机启动
sudo systemctl enable nfs-server.service

# 启动服务
systemctl start nfs-server

# 查看运行状态
systemctl status nfs-server
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/1081a94a-63fb-4181-82ee-5021a4fa0b6a)

- 创建共享目录，并赋值读写权限(master节点)
```
# 创建一个共享目录
mkdir /mnt/nfs
chmod -R 777 /mnt/nfs
```
- 导出共享目录
```
cat >> /etc/exports << EOF
/mnt/nfs *(rw,sync,no_root_squash)  # /nfs/data *(rw,no_root_squash,sync,no_subtree_check)  # 新版nfs
EOF
```
如果要限制网段则：
```
cat >> /etc/exports << EOF
# 限制为一个网段中
/mnt/nfs 192.168.1.0/24(rw,sync,all_squash)
EOF
```
参数说明：

  1. /mnt/nfs：共享目录的路径。任何连接到此NFS服务器的客户端都将访问此目录
  2. `*`: 通配符，表示允许任何IP地址访问
  3.  rw : 这表示读写权限。这意味着客户端用户该目录的读写权限
  4.  sync： 表示同步写入，即数据完全写入磁盘后才返回写入操作的响应
  5.  no_root_squash : 这表示允许root用户在远程机器上访问NFS时具有root权限。默认情况下，NFS会对root用户的请求进行“squash”，使其权限降低。通过设置 no_root_squash，可以允许root用户具有与NFS服务器上的root用户相同的权限。

- 重启服务或重新加载配置
```
systemctl restart nfs-server
# 使得配置生效
exportfs -r
# 查看生效
exportfs 
```
- 启动rpcbind
```
systemctl restart rpcbind   (重启)
systemctl enable rpcbind    (设置为开机自启动)
systemctl status rpcbind  (查看状态，验证重启成功)
```
- 验证rpcbind、nfs
```
# 查看rpc服务的注册情况
rpcinfo -p localhost
# showmount测试   　　
# showmount命令用于查询NFS服务器的相关信息   　　-e或--exports  显示NFS服务器的输出清单。
showmount -e 192.168.1.21
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/1758dcea-7643-4bdd-b5c0-7975709c467e)

#### 2.5.2 客户端配置挂载目录（所有node节点）
- 安装客户端
```
yum install nfs-utils
```
- 验证是否可以挂载目录
```
showmount -e 192.168.1.19
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/416c3544-80e8-44e4-91c0-d1235dec95bd)

- 创建/mnt/nfs_client 目录挂载到远程服务器
```
mkdir -p /mnt/nfs_client
mount -t nfs 192.168.1.21:/mnt/nfs /mnt/nfs_client
# 或者
mount -t nfs -o nolock 192.168.1.21:/mnt/nfs /mnt/nfs_client
```
服务器上的`/mnt/nfs`目录则挂载到了本地的`/mnt/nfs_client`目录。
说明：
1. mount：挂载文件系统或网络共享。
2. -t nfs：指定使用NFS（网络文件系统）协议。
3. -o nolock：指定挂载选项，其中nolock表示不启用文件锁定。
4. server:/mnt/nfs：指定远程服务器的挂载源，即服务器上的/mnt/nfs目录。
5. /mnt/nfs_client：指定本地挂载点，即在本地的/mnt/nfs_client目录下挂载。

需要永久挂载则编辑`/etc/fstab`文件。加入：
```
192.168.1.21:/mnt/nfs /mnt/nfs_client nfs defaults 0 0
```
- 查看挂载点是否挂载成功
```
df -h
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/e62f12d4-ec06-46af-bc36-52c807e380e8)
