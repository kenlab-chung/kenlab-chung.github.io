# OpenSSL自签证书
## 1 OpenSSL自签证书
- 创建http.ext文件
需要指定要访问的IP，内容如下：
```
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName=@SubjectAlternativeName

[SubjectAlternativeName]
IP.1=127.0.0.1
IP.2=192.168.1.2
```
- 生成密钥
```
openssl req -new -newkey rsa:2048 -sha256 -nodes -out 192.168.1.2.csr -keyout 192.168.1.2.key -subj "/C=CN/ST=Beijing/L=Beijing/O=Super Inc./OU=Web Security/CN=192.168.1.2"
```
- 生成证书
```
openssl x509 -req -days 3650 -in 192.168.1.2.csr -signkey 192.168.1.2.key -out 192.168.1.2.crt -extfile http.ext
```
-  将证书打包为pkcs12格式
```
openssl pkcs12 -export -in 192.168.1.2.crt -inkey 192.168.1.2.key -out 192.168.1.2.p12 -name bsoft-media
```
- 导出pem格式证书
```
openssl pkcs12 -in 192.168.1.2.p12 -out 192.168.1.2.pem -nodes
```
运行此命令后，您将被要求输入 .p12 文件的密码。然后，OpenSSL 将提取证书和私钥，并将其保存到 .pem 文件中。 -nodes 选项表示不加密私钥。
## 2 在本地谷歌浏览器安装证书
在谷歌浏览器[设置]->[隐私和安全]->[安全]->[证书管理]导入证书

![image](https://github.com/user-attachments/assets/51e9c45a-65b2-444c-9f0b-f9ade8b6039a)

点击[导入]导入证书

![image](https://github.com/user-attachments/assets/7b3083ac-8df3-40c4-977a-d4e4f4a59e06)

清除浏览器缓存

![image](https://github.com/user-attachments/assets/3b632d7d-8d53-4a99-8e19-524afc572757)

最后把国标平台重新加载192.168.1.2.p12证书文件

## 3 访问
![image](https://github.com/user-attachments/assets/75379ceb-3be3-4006-9e82-a8892983e0e0)

 链接是安全的：

 ![image](https://github.com/user-attachments/assets/f1cd3689-2ded-44a7-a490-ccaf2f1c695a)
 
