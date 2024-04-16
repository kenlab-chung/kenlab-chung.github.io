# K8s集群部署指南
## 1 环境

|       IP       |   操作系统  |   主机名称  |       配置        |    网络 |
|----------------|-------------|-------------|-------------------|---------|
|  192.168.1.110 |  CentOS7.9  |  k8s-master | 4vCPU/2G内存/20G硬 | 桥接模式 |
|  192.168.1.111 |  CentOS7.9  |  k8s-node01 | 4vCPU/2G内存/20G硬 | 桥接模式 |
|  192.168.1.112 |  CentOS7.9  |  k8s-node02 | 4vCPU/2G内存/20G硬 | 桥接模式 |

## 2 操作系统配置
分别修改三台主机名称
```
# 设置 192.168.1.110 主机名
hostnamectl set-hostname  k8s-master
# 设置 192.168.1.111 主机名
hostnamectl set-hostname  k8s-node01
# 设置 192.168.1.112 主机名
hostnamectl set-hostname  k8s-node02
```
