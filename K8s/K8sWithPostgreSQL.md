# k8s集群部署PostgreSQL
## 1 前置条件
- 操作系统环境：CentOS 7.9
- Docker版本：v18.06.3
- K8s版本：v1.23
- K8s部署，参考[K8s集群部署指南](./K8sClusterDeployment.md)
## 2 Helm
### 2.1 安装Helm
Helm提供了一种在Kubernetes集群上部署PostgreSQL实例的快速简便的方法。
```
wget https://get.helm.sh/helm-v3.14.4-linux-amd64.tar.gz
tar -xzvf helm-v3.8.0-linux-amd64.tar.gz
cp linux-amd64/helm /usr/local/bin/
helm version
```
### 2.2 添加Helm存储库
在 [Artifact Hub](https://artifacthub.io/) 中搜索要使用的 PostgreSQL Helm 图表。
```
helm search hub postgresql
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/30c0ec82-413f-4544-b7e5-846cc7535cba)

将 charts 的存储库添加到本地 Helm(添加 Bitnami Helm chart，添加存储库后，更新本地存储库)。
```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```
查看
```
helm repo list
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/2a5e3dd7-dfd7-4ece-b86a-325e1fefdbe7)
## 3 持久存储安装PostgreSQL
### 3.1 创建 PersistentVolume
Postgres 数据库中的数据需要在 pod 重新启动后保持不变。
```
cat <<EOF> postgres-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgresql-pv
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
EOF
```
```
kubectl apply -f postgres-pv.yaml
```
### 3.2 创建 PersistentVolumeClaim
```
cat <<EOF> postgres-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgresql-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
EOF
```
```
kubectl apply -f postgres-pvc.yaml
```
使用kubectl get检查PVC是否连接到PV成功：
```
kubectl get pvc
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/121175e6-f044-41b8-b728-2f1622a59b79)


