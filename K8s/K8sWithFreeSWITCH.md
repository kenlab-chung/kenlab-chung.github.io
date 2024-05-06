# K8s部署软交换集群
## 1 环境
前置条件：K8s集群部署完毕
|       IP       |   操作系统  |   主机名称  |       配置        |    网络 |
|----------------|-------------|-------------|-------------------|---------|
|  192.168.1.28  |  CentOS7.9  |  k8s-master | 4vCPU/2G内存/20G硬 | 桥接模式 |
|  192.168.1.42  |  CentOS7.9  |  k8s-node01 | 4vCPU/2G内存/20G硬 | 桥接模式 |
|  192.168.1.34  |  CentOS7.9  |  k8s-node02 | 4vCPU/2G内存/20G硬 | 桥接模式 |

##  2 FreeSWITCH部署
### 2.1 加载FreeSWITCH镜像(master节点)
- 加载镜像
```
 docker load -i docker-bsoft-fs-x64-v1.0.2.tar
```
- 查看镜像
```
docker images
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/00509da9-c9ca-47b7-baa0-6e46a5a181cc)

### 2.2 部署FreeSWITCH镜像
- 编写部署脚本
```
cat > freeswitch-deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bsoft-switch
spec:
  replicas: 3
  selector:
    matchLabels:
      app: bsoft-switch
  template:
    metadata:
      labels:
        app: bsoft-switch
    spec:
      containers:
      - image: bsoft-switch:v1.0.2
        name: bsoft-switch

---
apiVersion: v1
kind: Service
metadata:
  name: bsoft-switch
spec:
  selector:
    app: bsoft-switch
  ports:
    - protocol: TCP
      port: 5060
      targetPort: 5060
      nodePort: 31100
  type: NodePort
EOF
```
- 部署FS集群
```
kubectl apply -f freeswitch-deployment.yaml
```
  
