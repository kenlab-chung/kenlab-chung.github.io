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

- 推送镜像(测试)
```
#先登录
docker login 192.168.1.28:5000
#推送镜像
docker tag registry:2.6.2 192.168.1.28:5000/registry:2.6.2
docker push 192.168.1.28:5000/registry:2.6.2
#查看镜像
curl -u freeswitch:freeswitch 192.168.1.28:5000/v2/_catalog
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/4aa235b1-d802-4a47-870b-01551c70816a)

##  3 FreeSWITCH部署(master节点)
### 3.1 加载FreeSWITCH镜像
- 加载镜像
```
 docker load -i docker-bsoft-fs-x64-v1.0.2.tar
```
- 查看镜像
```
docker images
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/00509da9-c9ca-47b7-baa0-6e46a5a181cc)

- 推送freeSWITCH镜像
```
curl -u freeswitch:freeswitch 192.168.1.28:5000/v2/_catalog
#先登录
docker login 192.168.1.28:5000
#推送镜像
docker tag bsoft-switch:v1.0.2 192.168.1.28:5000/bsoft-switch:v1.0.2
docker push 192.168.1.28:5000/bsoft-switch:v1.0.2
#查看镜像
curl -u freeswitch:freeswitch 192.168.1.28:5000/v2/_catalog
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/9f4cebac-4da9-469c-9db3-ada683013877)

- K8s创建密钥
```
#创建freeswitch命名空间
kubectl create namespace freeswitch
kubectl create secret docker-registry registry-secret-name --docker-server=192.168.1.28:5000 --docker-username=freeswitch --docker-password=freeswitch -n freeswitch
```
- 查看当前运行的Pod和Service
```
kubectl get pods,service
```
### 3.2 部署FreeSWITCH集群
- 编写deployment文件
```
cat > freeswitch-deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bsoft-switch
  namespace: freeswitch
spec:
  replicas: 2
  selector:
    matchLabels:
      run: bsoft-switch
  template:
    metadata:
      labels:
        run: bsoft-switch
    spec:
      containers:
        - name: bsoft-switch
          image: 192.168.1.28:5000/bsoft-switch:v1.0.2 #修改为私有仓库的image
          volumeMounts:
            - name: host-time
              mountPath: /etc/localtime
          ports:
            - containerPort: 5060
          resources:
            requests:
              cpu:  1
              memory: 1024Mi
            limits:
              cpu:  1
              memory: 1024Mi

      imagePullSecrets:
        - name: registry-secret-name #为刚刚创建的密钥
      volumes:
        - name: host-time
          hostPath:
            path: /etc/localtime
---
apiVersion: v1
kind: Service
metadata:
  name: bsoft-switch
  namespace: freeswitch
  labels:
    run: bsoft-switch
spec:
  type: NodePort
  ports:
    - port: 5060
  selector:
    run: bsoft-switch
EOF
```
- 创建Deployment
```
kubectl apply -f freeswitch-deployment.yaml
kubectl get pods,service -n freeswitch
```
![image](https://github.com/kenlab-chung/kenlab-chung.github.io/assets/59462735/f29196f1-67be-4a78-a8cf-e0f5df64b54f)

