# K8s部署MySQL主从结构
## 1 需求与方案
### 1.1 需求
- 启动顺序有要求，master节点必须比slave节点先启动 （statefulset）
- 节点宕机，新的pod启动必须使用原先pod的资源（持久化保证数据不丢失）
- master与slave配置不一样
- master启动之后需要设置主从授权账户，slave需要执行change master命令，以及加入主从的命令
