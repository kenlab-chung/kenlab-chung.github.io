# OpenSSL自签证书
## OpenSSL自签证书
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





  
