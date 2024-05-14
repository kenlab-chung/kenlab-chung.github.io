#  通过Heketi管理GlusterFS为K6s集群提供持久化存储
## 1. Heketi简介
Heketi是一个提供RESTful API管理GlusterFS卷的框架，便于管理员对GlusterFS进行操作：

- 可以用于管理GlusterFS卷的生命周期;
- 能够在OpenStack，Kubernetes，Openshift等云平台上实现动态存储资源供应（动态在GlusterFS集群内选择bricks构建volume）；
- 支持GlusterFS多集群管理。

 框架：

 ![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/3fea9dbc-3d7d-4d0d-9971-a0e2e029dd5a)
