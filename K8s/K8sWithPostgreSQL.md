# k8s集群部署PostgreSQL
## 1 前置条件
- 操作系统环境：CentOS 7.9
- Docker版本：v18.06.3
- K8s版本：v1.23
- K8s部署，参考[K8s集群部署指南](./K8sClusterDeployment.md)

## 2 手动安装PostgreSQL
### 2.1 创建 PersistentVolume
```
cat >> postgres-configmap.yaml << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  labels:
    app: postgres
data:
  POSTGRES_DB: postgresdb
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: postgres
EOF
kubectl apply -f postgres-configmap.yaml
```
### 2.2 创建 PersistentVolume & PersistentVolumeClaim
```
cat >> postgres-storage.yaml << EOF
kind: PersistentVolume
apiVersion: v1
metadata:
  name: postgres-pv-volume
  labels:
    type: local
    app: postgres
spec:
  storageClassName: manual
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/mnt/data"
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: postgres-pv-claim
  labels:
    app: postgres
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
EOF
kubectl apply -f postgres-storage.yaml
```
### 2.3 创建 PostgreSQL Deployment
```
cat >> postgres-deployment.yaml<< EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 2
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:10.1
          imagePullPolicy: "IfNotPresent"
          ports:
            - containerPort: 5432
          envFrom:
            - configMapRef:
                name: postgres-config
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgredb
      volumes:
        - name: postgredb
          persistentVolumeClaim:
            claimName: postgres-pv-claim
EOF
kubectl apply -f postgres-deployment.yaml
```
### 2.4 创建 PostgreSQL Service
```
cat >> postgres-service.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    app: postgres
spec:
  type: NodePort
  ports:
   - port: 5432
  selector:
   app: postgres
EOF
kubectl apply -f postgres-service.yaml
```
### 2.5 查看部署清单
```
kubectl get all
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/774ad313-7f64-46b5-89ae-6ede8c9d3fc4)

可以看到类型为`NodePort`的postgres服务在Kubernetes主机上为PostgreSQL客户端连接公开了访问端口31833。此次NodePort是随机选择的端口，NodePort服务会在30000-32767之间随机选择服务端口。
### 2.6 查看日志
```
kubectl logs postgres-5cb8b67d8f-m59v6
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/d87712a9-3942-4b57-8cca-7e26afbaa1f4)

### 2.7 kubectl 连接 PostgreSQL
```
kubectl exec -ti postgres-5cb8b67d8f-m59v6 -- psql -h localhost -U postgres --password -p 5432 postgresdb
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/0d2d56d7-f192-4774-a59c-4d2345ef5c1d)

### 2.8 客户端 psql 连接 PostgreSQL
获取节点IP地址
```
kubectl get no -owide
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/a07c42b0-8c36-43f6-acbb-fee5ffb8b396)
```
psql -h  192.168.1.22 -U postgres --password -p 31833 postgresdb
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/53be7f5a-ecab-4b03-8042-8fc3b908860c)
