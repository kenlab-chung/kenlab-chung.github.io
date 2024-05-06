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
- 建立Docker私有仓库
拉取镜像
```
docker pull registry
```
启动镜像
```
docker run -d -p 5000:5000 -v /myregistry:/var/lib/registry registry
```

- 加载镜像
```
 docker load -i docker-bsoft-fs-x64-v1.0.2.tar
```
- 查看镜像
```
docker images
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/00509da9-c9ca-47b7-baa0-6e46a5a181cc)

- 修改镜像tag名称
```
docker tag bsoft-switch:v1.0.2 192.168.1.28:5000/bsoft-switch:v1.0.2
```
- 将镜像192.168.1.28:5000/bsoft-switch:v1.0.2上传到本地仓库
```
docker push 192.168.1.28:5000/bsoft-switch:v1.0.2
```
默认情况下本地仓库默认使用https协议进行上传。所以出现如下错误

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/fad2f4e0-dc38-4cbe-8e70-a1916fe16830)

此时可以修改/usr/lib/systemd/system/docker.service中的参数，使docker支持非https协议上传。即在ExecStart参数后面添加
```
--insecure-registry 192.168.1.28:5000
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/84cc4ab9-ad75-4fc6-a888-c84480d71499)

- 重启docker服务
```
systemctl daemon-reload
systemctl restart docker
```
- 重启registry容器
```
docker restart ef56f59efd78
```
- 再次上传192.168.1.28:5000/bsoft-switch:v1.0.2镜像
```
docker push 192.168.1.28:5000/bsoft-switch:v1.0.2
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/e63beebd-2a2a-49c8-b7d9-ba0407b4ecf2)


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
      - image: 192.168.1.28:5000/bsoft-switch:v1.0.2
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
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/5062ff7c-64f7-4f4e-a177-4495fa3372e9)

- 查看当前运行的Pod和Service
```
kubectl get pods,service
```

