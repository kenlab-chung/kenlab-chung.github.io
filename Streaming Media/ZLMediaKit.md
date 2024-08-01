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

