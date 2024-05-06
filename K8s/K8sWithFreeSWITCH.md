# K8s部署软交换集群
## 1 环境
前置条件：K8s集群部署完毕
|       IP       |   操作系统  |   主机名称  |       配置        |    网络 |
|----------------|-------------|-------------|-------------------|---------|
|  192.168.1.28  |  CentOS7.9  |  k8s-master | 4vCPU/2G内存/20G硬 | 桥接模式 |
|  192.168.1.42  |  CentOS7.9  |  k8s-node01 | 4vCPU/2G内存/20G硬 | 桥接模式 |
|  192.168.1.34  |  CentOS7.9  |  k8s-node02 | 4vCPU/2G内存/20G硬 | 桥接模式 |

## 2 搭建docker私有仓库
- 拉取镜像
```
docker pull registry:2.6.2
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/c0944993-9cfb-4e0f-980b-fdf33ff4c9d4)

- 创建一个文件夹用于存储用户名和密码，然后新创建一个账户
```
mkdir /var/auth
docker run --entrypoint htpasswd registry:2.6.2 -Bbn freeswitch freeswitch >/var/auth/htpasswd
cat /var/auth/htpasswd
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/c3937f05-a250-4584-8910-deb6bfd17702)

- 修改`/etc/docker/deamon.json`文件，添加如下内容。（所有节点）
```
  "insecure-registries":["192.168.1.28:5000"]
```
- 并重启docker。（所有节点）
```
systemctl daemon-reload
systemctl restart docker
```
- Registry服务默认将上传的镜像保存在容器的`/var/lib/registry`目录下，此时我们将主机`/opt/registry`目录挂载到该目录，即可实现将镜像保存到主机的`/opt/registry`目录了。然后将宿主机的`/var/auth`目录挂载到镜像的`/auth`目录下，然后指定这个目录下htpasswd文件来进行认证。
```
docker run -d -v /opt/registry:/var/lib/registry -v /var/auth:/auth -e "REGISTRY_AUTH=htpasswd" -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd  -p 5000:5000 --restart=always --name registry registry:2.6.2
```
> -p 5000:5000 指定registry的端口是5000，并映射成宿主机的5000端口。
> 
> -v /opt/registry:/var/lib/registry，将宿主机`/opt/registry`目录挂载到镜像默认存储路径`/var/lib/registry`。
> 
> -v /var/auth:/auth，将第二步生成auth文件夹挂在到镜像auth目录。
>
> -e REGISTRY_AUTH=htpasswd， -e REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm，这两个参数组合启动基本身份验证。
>
> -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd，指定使用的密码认证文件是/auth/htpasswd。(注意,使用的是容器里面的路径,前面我们已经将/var/auth挂在到/auth)
>
> 如果有https证书,可以加上以下参数:
>
> -v /usr/local/nginx/conf/cert:/certs，如果有https认证，将宿主机保存的认证文件挂到容器里。
>
> -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/server.pem，-e REGISTRY_HTTP_TLS_KEY=/certs/server.key 指定https证书和key。
>
> –restart=always，重启方式为always。
>
> –name registry，指定容器名称。
>
> registry:2.6.2，镜像名称+版本

![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/5e3e986c-5e26-40d0-808f-ac49364155a8)

##  2 FreeSWITCH部署
### 2.1 加载FreeSWITCH镜像(master节点)
- 建立Docker私有仓库

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

