# ZLMediaKit
## 一、环境
- debian 11 64bit
- ZLMediaKit 
- WVP-PRO
- MySQL 8
- 测试机IP:192.168.1.115
## 二、ZLMediaKit 编译
编译源码之前，先安装编译器、camke构建工具及第三方依赖库
```
# 安装编译器
sudo apt install build-essential cmake

# 其它依赖库
sudo apt-get install libssl-dev libsdl-dev libavcodec-dev libavutil-dev ffmpeg
```
完成后，直接拉去源码进行编译：
```
git clone https://github.com/ZLMediaKit/ZLMediaKit.git
cd ZLMediaKit

# 更新子模块
git submodule update --init
mkdir build
cd build
cmake ..
make -j4
```
编译好后，产生的可以执行文件放在如下目录：
```
./release/linux/Debug/
```
启动服务：
```
cd ../release/linux/debug

# 可以通过-h来查看命令支持的参数
sudo ./MediaServer 
```
成功启动后，控制台输出如下： 
![image](https://github.com/user-attachments/assets/c02991b6-faa9-4122-b0e9-5e9497e5650a)
接下来就可以在客户端进行推流。为了简单起见，笔者选择在同一台机器上推流。不同主机时，修改对应的IP地址即可：
- 使用rtsp方式推流
```
# h264编码
ffmpeg -re -i /home/debian/box.mp4 -vcodec h264 -acodec aac -f rtsp -rtsp_transport tcp rtsp://127.0.0.1/live/test
# h265编码
ffmpeg -re -i /home/debian/box.mp4 -vcodec h265 -acodec aac -f rtsp -rtsp_transport tcp rtsp://127.0.0.1/live/test
```
推流客户端输出如下：
![image](https://github.com/user-attachments/assets/cdfad40c-e349-477b-b857-77ad20309038)
服务器端输出如下：
![image](https://github.com/user-attachments/assets/f25c607f-0e40-41d1-9e99-45ac37741141)
可以看到几个媒体注册的消息，同时支持 rtsp、rtmp、hls等协议，现在可以另一台机器上开个播放器播放了，像vlc、ffplay 都可以，播放的url是：
```
rtsp://192.168.1.115/live/test
rtmp://192.168.1.115/live/test
http://192.168.1.115/live/test/hls.m3u8
http://192.168.1.115:9080/live/test.live.flv
```
![image](https://github.com/user-attachments/assets/383edca5-b5b7-4a79-a5d5-df76facd6715)

当然也可以用rtmp方式推，效果是一样的：
```
ffmpeg -re -i box.mp4 -c copy -f flv rtmp://127.0.0.1/live/test
```
## 三、安装MySQL数据库
安装MySQL数据库
```
wget https://dev.mysql.com/get/mysql-apt-config_0.8.26-1_all.deb
sudo apt install ./mysql-apt-config_0.8.26-1_all.deb
sudo apt update
sudo apt install mysql-server
```
创建wvp数据库
```
mysql -u root -p
CREATE DATABASE wvp;
```
## 四、安装redis数据库
```
sudo apt install redis
```
## 五、编译WVP-PRO
帮助文档参考：
```
https://doc.wvp-pro.cn/
```
拉取源码：
```
sudo git clone https://github.com/648540858/wvp-GB28181-pro.git
```
编译前端页面:
```
cd wvp-GB28181-pro/web_src/
npm --registry=https://registry.npmmirror.com install
npm run build
```
前端页面编译完成后如下图所示：

![image](https://github.com/user-attachments/assets/763765f3-fbd1-4ca4-9a76-700afd4fc5b5)

编译WVP-PRO程序
```
cd wvp-GB28181-pro
mvn package
```
编译成功后，在target目录下可以看到jar包。

![image](https://github.com/user-attachments/assets/ba973605-8ff3-4cdf-bdad-0f50f4df95f9)

配置数据库
```
use wvp;
set character_set_client=utf8;
set character_set_connection=utf8;
set character_set_database=utf8;
set character_set_results=utf8;
source /home/wvp-GB28181-pro/sql/初始化.sql;
```
执行脚步后会创建如下表结构：

![image](https://github.com/user-attachments/assets/eaeed4df-460c-4921-9dff-e70fcefb20f7)

修改yml配置文件
```
cp ./src/main/resources/application-dev.yml ./src/main/resources/application-local.yml
```
编辑 application-local.yml:
- 配置数据库
  
![image](https://github.com/user-attachments/assets/ed4654cb-c226-4b00-81d8-31491c0b2c40)

- 配置28181侦听地址
  
![image](https://github.com/user-attachments/assets/4929c2b3-74ab-4aec-9a1b-4177a768389b)

- 配置zml连接信息
其中id为ZLMediaKit的服务ID，必须配置，要与ZLMediaKit/release/linux/Debug/config.ini文件中mediaServerId一致。

![image](https://github.com/user-attachments/assets/a16585da-a75d-4d36-acf1-8dcab556cf3a)

- 配置wvp服务启动端口
  
![image](https://github.com/user-attachments/assets/797768e4-af8c-4eef-8406-3bf59a3550ac)



