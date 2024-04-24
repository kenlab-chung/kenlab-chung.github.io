# k8s集群部署PostgreSQL
## 1 前置条件
- 操作系统环境：CentOS 7.9
- Docker版本：v18.06.3
- K8s版本：v1.23
- K8s部署，参考[K8s集群部署指南](./K8sClusterDeployment.md)
## 2 安装Helm
Helm提供了一种在Kubernetes集群上部署PostgreSQL实例的快速简便的方法。
```
wget https://get.helm.sh/helm-v3.14.4-linux-amd64.tar.gz
tar -xzvf helm-v3.8.0-linux-amd64.tar.gz
cp linux-amd64/helm /usr/local/bin/
helm version
```
## 3 创建 imagePullSecrets
创建容器集群访问 uhub.service.ucloud.cn/ucloud_pts 需要的 secret
```
kubectl create namespace db
kubectl create secret docker-registry registry-secret-name \
        --namespace=db \
        --docker-server=uhub.service.ucloud.cn/ucloud_pts \
        --docker-username='postgres' \
        --docker-password='postgres'
```
## 3 使用Helm部署postgresql
```
cat > pg-values.yaml << EOF
image:
  registry: uhub.service.ucloud.cn/ucloud_pts
  repository: postgresql
  tag: 13.3.0-debian-10-r55
global:
  imagePullSecrets:
    - registry-secret-name
  storageClass: csi-udisk-rssd
  postgresql:
    postgresqlDatabase: dbname
    postgresqlUsername: pgadmin
    postgresqlPassword: passwdxxxx
EOF
helm install gitlib-db -f pg-values.yaml bitnami/postgresql -n db
```
输出如下：
```
NAME: gitlib-db
LAST DEPLOYED: Wed Apr 17 18:00:51 2024
NAMESPACE: db
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: postgresql
CHART VERSION: 15.2.5
APP VERSION: 16.2.0

** Please be patient while the chart is being deployed **

PostgreSQL can be accessed via port 5432 on the following DNS names from within your cluster:

    gitlib-db-postgresql.db.svc.cluster.local - Read/Write connection

To get the password for "postgres" run:

    export POSTGRES_PASSWORD=$(kubectl get secret --namespace db gitlib-db-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)

To connect to your database run the following command:

    kubectl run gitlib-db-postgresql-client --rm --tty -i --restart='Never' --namespace db --image uhub.service.ucloud.cn/ucloud_pts/postgresql:13.3.0-debian-10-r55 --env="PGPASSWORD=$POSTGRES_PASSWORD" \
      --command -- psql --host gitlib-db-postgresql -U postgres -d postgres -p 5432

    > NOTE: If you access the container using bash, make sure that you execute "/opt/bitnami/scripts/postgresql/entrypoint.sh /bin/bash" in order to avoid the error "psql: local user with ID 1001} does not exist"

To connect to your database from outside the cluster execute the following commands:

    kubectl port-forward --namespace db svc/gitlib-db-postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432

WARNING: The configured password will be ignored on new installation in case when previous PostgreSQL release was deleted through the helm command. In that case, old PVC will have an old password, and setting it through helm won't take effect. Deleting persistent volumes (PVs) will solve the issue.

WARNING: There are "resources" sections in the chart not set. Using "resourcesPreset" is not recommended for production. For production installations, please set the following values according to your workload needs:
  - primary.resources
  - readReplicas.resources
+info https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/
```
如果需要部署不同PG版本，选择同步不同版本镜像即可：
- PG v13: bitnami/postgresql:13.3.0-debian-10-r55
- PG v12: bitnami/postgresql:12.7.0-debian-10-r51
- PG v11: bitnami/postgresql:11.12.0-debian-10-r53
  
更多版本可以参考
```
https://hub.docker.com/r/bitnami/postgresql/tags?page=1&ordering=last_updated&name=debian
```
## 4 验证部署
