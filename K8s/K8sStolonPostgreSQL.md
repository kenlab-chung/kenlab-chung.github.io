# k8s Stolon 部署高可用PostgreSQL
## 1 前置条件
- 操作系统环境：CentOS 7.9
- Docker版本：v18.06.3
- K8s版本：v1.23
- K8s部署，参考[K8s集群部署指南](./K8sClusterDeployment.md)
## 2 PostgreSQL高可用方案选型
- 首先repmgr这种方案的算法有明显缺陷，非主流分布式算法，直接排除。
- Stolon和Patroni相对于Crunchy更加Cloud Native， 后者是基于pgPool实现。
- Crunchy和Patroni相对于Stolon有更多的使用者，并且提供了Operator对于以后的管理和扩容。
根据上面简单的比较，最终选择的stolon。
## 3 Stolon概述
Stolon是由3个部分组成：
1. keeper：他负责管理PostgreSQL的实例汇聚到由sentinel(s)提供的clusterview。
2. sentinel：it负责发现并且监控keeper，并且计算最理想的clusterview。
3. proxy：客户端的接入点。它强制连接到右边PostgreSQL的master并且强制关闭连接到由非选举产生的master。

Stolon 用etcd或者consul作为主要的集群状态存储。
## 4 安装
- 下载安装文件
```
git clone https://github.com/sorintlab/stolon.git
```
- 设置Postgresql用户名(stolon-keeper.yaml文件)
```
  - name: STKEEPER_PG_SU_USERNAME
            value: "postgres"
```

- 设置用户密码(stolon-keeper.yaml文件)
```
apiVersion: v1
kind: Secret
metadata:
    name: stolon
type: Opaque
data:
    password: postgres
```
- 设置stolon挂载卷(stolon-keeper.yaml文件)
 ```
    volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes:
        - "ReadWriteOnce"
      resources:
        requests:
          storage: "512Mi"
      storageClassName: nfs
   ```
