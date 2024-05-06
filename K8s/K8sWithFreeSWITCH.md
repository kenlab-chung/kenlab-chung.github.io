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
```
 docker load -i docker-bsoft-fs-x64-v1.0.2.tar
```
