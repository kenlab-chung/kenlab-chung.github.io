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
tar -xzvf helm-v3.14.4-linux-amd64.tar.gz
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
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/e41d43b6-5a37-4418-8ce0-7fa293e5f707)

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
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/45d78861-cd0b-4a4c-8e3d-e9299a829896)

### 3.3 安装PostgreSQL
```
helm install psql-test bitnami/postgresql --set persistence.existingClaim=postgresql-pv-claim --set volumePermissions.enabled=true
```
输出
```
NAME: psql-test
LAST DEPLOYED: Wed Apr 24 09:53:37 2024
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: postgresql
CHART VERSION: 15.2.5
APP VERSION: 16.2.0

** Please be patient while the chart is being deployed **

PostgreSQL can be accessed via port 5432 on the following DNS names from within your cluster:

    psql-test-postgresql.default.svc.cluster.local - Read/Write connection

To get the password for "postgres" run:

    export POSTGRES_PASSWORD=$(kubectl get secret --namespace default psql-test-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)

To connect to your database run the following command:

    kubectl run psql-test-postgresql-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:16.2.0-debian-12-r15 --env="PGPASSWORD=$POSTGRES_PASSWORD" \
      --command -- psql --host psql-test-postgresql -U postgres -d postgres -p 5432

    > NOTE: If you access the container using bash, make sure that you execute "/opt/bitnami/scripts/postgresql/entrypoint.sh /bin/bash" in order to avoid the error "psql: local user with ID 1001} does not exist"

To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace default svc/psql-test-postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432

WARNING: The configured password will be ignored on new installation in case when previous PostgreSQL release was deleted through the helm command. In that case, old PVC will have an old password, and setting it through helm won't take effect. Deleting persistent volumes (PVs) will solve the issue.

WARNING: There are "resources" sections in the chart not set. Using "resourcesPreset" is not recommended for production. For production installations, please set the following values according to your workload needs:
  - primary.resources
  - readReplicas.resources
  - volumePermissions.resources
+info https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
```
查看状态
