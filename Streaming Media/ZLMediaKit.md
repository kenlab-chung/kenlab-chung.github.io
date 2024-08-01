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


