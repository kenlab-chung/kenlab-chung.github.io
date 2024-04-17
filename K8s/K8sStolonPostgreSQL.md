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

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/4be69dbb-3b1e-4493-96a1-a026d58e9aad)

- 生成密码
```
# 因为Secret中只能保存base64格式的密码，所以生成一个base64的密码
[root@k8s-master ~]# echo -n "postgres"  | base64
cG9zdGdyZXM=
```
- 修改密码(secret.yaml)
```
apiVersion: v1
kind: Secret
metadata:
    name: stolon
type: Opaque
data:
    password: cG9zdGdyZXM=
```
- 使用NodePort的方式对外提供服务
```
# 修改 stolon-proxy-service.yaml文件
apiVersion: v1
kind: Service
metadata:
  name: stolon-proxy-service
spec:
  ports:
    - port: 5432
      targetPort: 5432
      nodePort: 31000  # 指定端口
  type: NodePort       # 类型为NodePort
  selector:
    component: stolon-proxy
    stolon-cluster: kube-stolon
```
- 执行yaml文件
  ``` 
  kubectl apply -f .
  ```
- 初始化数据库
  ```
  kubectl run -i -t stolonctl --image=sorintlab/stolon:master-pg10 --restart=Never --rm -- /usr/local/bin/stolonctl --cluster-name=kube-stolon --store-backend=kubernetes --kube-resource-kind=configmap init
  ```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/566ff276-4c99-4826-9539-b87d1962a53d)
  
- 删除PostgreSQL数据库
  ```
  kubectl delete -f stolon.yaml
  kubectl delete pvc data-stolon-keeper-0 data-stolon-keeper-1
  ```

  
