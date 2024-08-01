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

- 修改ZLMediaKit配置
```
cd ZLMediaKit/release/linux/Debug
vim config.ini
```
修改mediaServerId，与 wvp中的media.id保持一致。

![image](https://github.com/user-attachments/assets/12153eff-7a41-4abc-a43e-202f84f94eb6)

- 修改http端口和sslport端口
 
![image](https://github.com/user-attachments/assets/28a5afe6-0da9-45a1-ae47-9dd30202dbc6)

在wvp-GB28181-pro目启动项目：
```
mvn spring-boot:run
```
上述启动方法

优点：方便修改配置

确定：启动慢，理论上不影响性能。

当然可以在target目录直接启动jar包
```
java -jar wvp-pro-2.6.9-08180706.jar
或
java -jar wvp-pro-*.jar --spring.config.location=../src/main/resources/application.yml
```
## 六、运行
```
#启动ZLM
./MediaServer

#启动WVP
cd wvp-GB28181-pro/target
java -jar wvp-pro-*.jar
```
成功运行后，在浏览器中打开你的ip地址+WVP监听的HTTP端口（例如：192.168.1.115:18080）即可看到登录界面

![image](https://github.com/user-attachments/assets/967fe24d-9566-4db2-b855-4aa4b2cea96a)

用户名密码为admin/admin

登录后，界面如下：

![image](https://github.com/user-attachments/assets/c2318d4d-4814-48cb-8030-367b328465cc)

## 七、接入海康硬盘录像机NVR
在海康硬盘录像机NVR平台接入选项卡下配置mvp信息

![image](https://github.com/user-attachments/assets/0a7f6afa-66f2-463a-bd76-10ba81f31c17)

上述配置信息与wvp 的28181服务器配置一致： 

![image](https://github.com/user-attachments/assets/c5aa7d46-f041-4365-9bdf-4f0f3ff327aa)

重启NVR后，可以在wvp web管理界面中看到视频

![image](https://github.com/user-attachments/assets/8677b9b5-d7e7-4d6f-80ef-51ca1e23b641)

同时在zml控制台上可以看到媒体注册输出：

![image](https://github.com/user-attachments/assets/1c4f6107-d4af-4e74-9f94-bafe78afb78a)

用rtsp测试：
```
rtsp://192.168.1.115:554/rtp/34020000001110000001_34020000001310000001
```
用vlc网络串流测试，成功打开画面：

![image](https://github.com/user-attachments/assets/55932c03-ccae-44a1-808c-4ce5fe379933)

## 八、FreeEHome编译
```
git clone https://github.com/tsingeye/FreeEhome.git
cd FreeEhome
go env -w GOPROXY=https://goproxy.cn
go get
go build
```
## 九、对接海康ISUP(eHome 协议)
- 下载海康ISUP SDK
```
https://open.hikvision.com/download/5cda567cf47ae80dd41a54b3?type=10&id=18e1e779efed4593bfceba6703d7f6a8
```
选择ISUP的Linux64位下载：

![image](https://github.com/user-attachments/assets/17af57d5-ea38-4b2e-b6fc-370084136a96)

-  编译ffmpeg
安装依赖库
```
sudo apt install -y libx264-dev libx265-dev libass-dev libmp3lame-dev libopus-dev libsdl2-dev python gcc cmake make p7zip-full vim pkg-config autoconf automake build-essential nasm yasm dos2unix
```
下载ffmpeg 6.0
```
https://ffmpeg.org/download.html
```
编译ffmpeg 6.0
```
./configure --enable-shared --enable-static --enable-libx264  --enable-gpl  
make -j4
sudo make install
```
分别查看ffmpeg、ffplay、ffprobe版本
```
ffmpeg -version
ffplay -version
ffprobe -version
```
cmakelists.txt中引用静态库
```
CMAKE_MINIMUM_REQUIRED(VERSION 3.18)

PROJECT(demo)
add_compile_options(-std=c++20)
set(CMAKE_BUILD_TYPE "Debug")
set(CMAKE_CXX_FLAGS_DEBUG "$ENV{CXXFLAGS} -O0 -Wall -g -ggdb -std=c++20")
set(CMAKE_CXX_FLAGS_RELEASE "$ENV{CXXFLAGS} -O3 -Wall -std=c++20")
add_definitions(-std=c++20)

INCLUDE_DIRECTORIES(inc)
LINK_DIRECTORIES(./lib)
ADD_EXECUTABLE(demo ./src/main.cpp)

TARGET_LINK_LIBRARIES(demo
     libHCISUPCMS.so
     libHCISUPStream.so
     libHCISUPSS.so
     libcrypto.so
     libcrypto.so.1.0.0
     libHCISUPAlarm.so
     libHCISUPSS.so
     libHCNetUtils.so
     libhpr.so
     libNPQos.so
     libsqlite3.so
     libssl.so
     libssl.so.1.0.0
     libz.so
     libavdevice.a
     libavformat.a
     libavutil.a
     )
```
main.cpp 源码文件中引用头文件：
```
#include <stdio.h> 
extern "C" {
#include <libavcodec/avcodec.h>
}

int main()
{
        auto aversion = av_version_info();
        printf("ffmpeg version %s!\n",aversion); 
}
```
在回调ISUP回调中将数据写入管道
```
const string pipe_name = "/tmp/test.ipc";
BOOL InputStreamData(BYTE byDataType, char* pBuffer, int iDataLen)
{
    if(iDataLen<=40)
    { 
        if ((pBuffer[0] == 0x49)
            && pBuffer[1] == 0x4D 
            && pBuffer[2] == 0x4B 
            && pBuffer[3] == 0x48
            ) 
            {
                umask(0); 
                _fd = open(pipe_name.c_str(),O_WRONLY); 
            }                                                                                                                                                                
     }
     else   
     {  
        write(_fd,pBuffer,iDataLen); 
     }   
}
```
接下来采用ffmpeg将数据推送到ZLM，伪代码如下：
```
const char* out_filename ="rtmp://127.0.0.1/live/stream";
static double _r2d(AVRational r)
{
    return r.den == 0 ? 0 : (double) r.num / (double) r.den;
}
void runPipe()
{
    int ret,i;
    int video_index = -1;
    int frame_index = 0;
    int64_t start_time=0;

    avformat_network_init();
    AVOutputFormat *ofmt = nullptr;
    AVFormatContext *ofmt_ctx=nullptr;
    AVFormatContext * ic = nullptr;

    AVDictionary *option = nullptr;
    const char* pipe_name = "/tmp/bsoft.ipc";
    remove(pipe_name.c_str()); 
    int nRet = mkfifo(pipe_name.c_str(),0666);   
    if(nRet == -1) 
    {  
        printf("create bsoft.ps Error.\n");
        return;        
    }  
 
    av_dict_set(&option,"buffer_size","10240",0);
    if(avformat_open_input(&ic,pipe_name,nullptr,&option)!=0)
    {
        printf("open pipe failed %s\n",pipe_name);
        return;
    }
    printf("avformat_open_input() called success\n");

    printf("duration is %ld,nb_streams is:%d\n",ic->duration,ic->nb_streams);
    if(avformat_find_stream_info(ic,0)>=0)
    {
        printf("duration is %ld,nb_streams is:%d\n",ic->duration,ic->nb_streams);
    }
    else
    {
        printf("avformat_find_stream_info failed.\n");
        return;
    }
    int videoStream = av_find_best_stream(ic,AVMEDIA_TYPE_VIDEO,-1,-1,nullptr,0);
    int audioStream = av_find_best_stream(ic,AVMEDIA_TYPE_AUDIO,-1,-1,nullptr,0);

    AVPacket* pkt = av_packet_alloc();
    int nPacketIndex = 0;

    AVBSFContext* absCtx = nullptr;
    const AVBitStreamFilter * absFilter = nullptr;
    AVCodecParameters* codecpar = nullptr;

    absFilter = av_bsf_get_by_name("h264_mp4toannexb");

    if(!absFilter)
    {
        printf("get bsfilter fialed.\n");
        return;
    }

    ret = av_bsf_alloc(absFilter,&absCtx);
    if(ret!=0)
    {
        printf("av_bsf_alloc fialed.\n");
        return;
    }
    codecpar = ic->streams[videoStream]->codecpar;

    avcodec_parameters_copy(absCtx->par_in,codecpar);
    ret = av_bsf_init(absCtx);
    if(ret!=0)
    {
        printf("av_bsf_init fialed.\n");
        return;
    }


    av_dump_format(ic,0,pipe_name,0);

    avformat_alloc_output_context2(&ofmt_ctx,nullptr,"flv",out_filename);

    if(!ofmt_ctx)
    {
            printf("Failed to create output context.\n");
            goto end;
    }

    ofmt = (AVOutputFormat*)ofmt_ctx->oformat;

    for(i = 0;i<ic->nb_streams;i++)
    {
        AVStream * in_stream =  ic->streams[i];
        AVStream * out_stream = avformat_new_stream(ofmt_ctx,nullptr);
        AVCodecParameters *in_codecpar = in_stream->codecpar;
   
        if(!out_stream)
        {
            printf("Failed to allocate output stream.\n");
            goto end;
        }

        ret = avcodec_parameters_copy(out_stream->codecpar,in_codecpar);
        if(ret < 0)
        {
            printf("Failed to copy codec parameters.\n");
            goto end;
        }
        out_stream->codecpar->codec_tag = 0;

    }
    for(i = 0;i<ic->nb_streams;i++)
    {
        if(ic->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_VIDEO)
        {
            video_index = i;
            break;
        }
    }
    for(i = 0;i<ofmt_ctx->nb_streams;i++)
    {
        AVCodecParameters *codeParams = ofmt_ctx->streams[i]->codecpar;
        if(codeParams->codec_type == AVMEDIA_TYPE_AUDIO)
        {
            codeParams->sample_rate = 44100;
            codeParams->codec_id = AV_CODEC_ID_AAC;
            codeParams->bit_rate = 64000;
            //codeParams->channel_layout = AV_CH_LAYOUT_STEREO;
            break;
        }
    }
    av_dump_format(ofmt_ctx,0,out_filename,1);

    ret = avio_open(&ofmt_ctx->pb,out_filename,AVIO_FLAG_WRITE);
    if(ret < 0)
    {
        printf("Could not open output url.\n");
        goto end;
    }

    ret = avformat_write_header(ofmt_ctx,nullptr);
    if(ret < 0)
    {
        printf("Error occurred when opening output url .\n");
        goto end;
    }

    AVPacket pkt2;
    for(;;)
    {
        AVStream * in_stream=nullptr,* out_stream = nullptr;
        ret = av_read_frame(ic,pkt);
        if(ret != 0)
        {
            printf("==============end============\n");
            break;
        }

        nPacketIndex++;
        std::cout<<"index:"<<nPacketIndex;
        std::cout<<" bytes:"<<pkt->size;
        std::cout<<" pts:"<<pkt->pts;
        std::cout<<" dts:"<<pkt->dts;
        std::cout<<" num:"<<ic->streams[pkt->stream_index]->time_base.num;
        std::cout<<" den:"<<ic->streams[pkt->stream_index]->time_base.den;
        std::cout<<" pts-ms:"<<pkt->pts*(_r2d(ic->streams[pkt->stream_index]->time_base)*1000);
        if(pkt->stream_index == videoStream)
        {
            std::cout<<" Video"<<std::endl;
#if 1
            ret = av_bsf_send_packet(absCtx,pkt);
            if(ret!=0)
            {
                printf("av_bsf_init fialed.\n");
                return;
            }
            while(1)
            {
                ret = av_bsf_receive_packet(absCtx,&pkt2);
                if(ret == AVERROR(EAGAIN)||ret ==AVERROR_EOF)
                    break;

                //if(pkt2.pts == AV_NOPTS_VALUE)
                {
                    AVRational time_base1=ic->streams[video_index]->time_base;
                    int64_t calc_duration=(double)AV_TIME_BASE/av_q2d(ic->streams[video_index]->r_frame_rate);
                    pkt2.pts=(double)(frame_index*calc_duration)/(double)(av_q2d(time_base1)*AV_TIME_BASE);
                    pkt2.dts=pkt2.pts;
                    pkt2.duration=(double)calc_duration/(double)(av_q2d(time_base1)*AV_TIME_BASE);
                }
                if(pkt2.stream_index==video_index)
                {
                    AVRational time_base=ic->streams[video_index]->time_base;
                    AVRational time_base_q={1,AV_TIME_BASE};
                    int64_t pts_time = av_rescale_q(pkt2.dts, time_base, time_base_q);
                    int64_t now_time = av_gettime() - start_time;
                    if (pts_time > now_time)
                        av_usleep(pts_time - now_time);
                }

                in_stream  = ic->streams[pkt2.stream_index];
                out_stream = ofmt_ctx->streams[pkt2.stream_index];

                pkt2.pts = av_rescale_q_rnd(pkt2.pts, in_stream->time_base, out_stream->time_base, (AVRounding)(AV_ROUND_NEAR_INF|AV_ROUND_PASS_MINMAX));
                pkt2.dts = av_rescale_q_rnd(pkt2.dts, in_stream->time_base, out_stream->time_base, (AVRounding)(AV_ROUND_NEAR_INF|AV_ROUND_PASS_MINMAX));
                pkt2.duration = av_rescale_q(pkt2.duration, in_stream->time_base, out_stream->time_base);
                pkt2.pos = -1;
                if(pkt2.stream_index==video_index){
                    printf("Send %8d video frames to output URL\n",frame_index);
                    frame_index++;
                }
                //ret = av_write_frame(ofmt_ctx, &pkt);
                ret = av_interleaved_write_frame(ofmt_ctx, &pkt2);

                if (ret < 0) 
                {
                   printf( "Error muxing packet\n");
                   break;
                }
                av_packet_unref(&pkt2);
            }
#endif 
        }
        else if(pkt->stream_index == audioStream)
        {
            std::cout<<" audio"<<std::endl;
        }
        av_packet_unref(pkt);

    }

    av_packet_free(&pkt);

end:
    if(ic)
    {
        avformat_close_input(&ic);
    }

}
```
用vlc打开地址http://ip:port/live/stream.live.flv效果如图:

![image](https://github.com/user-attachments/assets/ca0f773c-7172-4ac6-a3b5-3e45914ebc18)

##  十、ffmpeg推送RTMP
```
extern "C" {
#include <libavformat/avformat.h>
#include <libavcodec/avcodec.h>
#include <libavutil/opt.h>
#include <libavutil/mathematics.h>
}

int main() {
    av_register_all();

    AVFormatContext *formatContext = NULL;
    AVOutputFormat *outputFormat = NULL;
    AVStream *videoStream = NULL;
    AVStream *audioStream = NULL;
    AVCodecContext *videoCodecContext = NULL;
    AVCodecContext *audioCodecContext = NULL;

    // 打开输出 RTMP URL
    if (avformat_alloc_output_context2(&formatContext, NULL, "flv", "rtmp://server_address/application/stream_name") < 0) {
        std::cerr << "Failed to allocate output context." << std::endl;
        return 1;
    }

    outputFormat = formatContext->oformat;

    // 添加视频流
    videoStream = avformat_new_stream(formatContext, NULL);
    if (!videoStream) {
        std::cerr << "Failed to create video stream." << std::endl;
        return 1;
    }

    // 添加音频流
    audioStream = avformat_new_stream(formatContext, NULL);
    if (!audioStream) {
        std::cerr << "Failed to create audio stream." << std::endl;
        return 1;
    }

    // 配置视频编码器和音频编码器参数
    // 这里假设您已经有视频和音频数据的 AVFrame 对象（videoFrame 和 audioFrame）
    // 请根据您的实际情况进行调整

    // 设置视频编码器参数
    videoCodecContext = videoStream->codec;
    videoCodecContext->codec_id = AV_CODEC_ID_H264;
    videoCodecContext->codec_type = AVMEDIA_TYPE_VIDEO;
    videoCodecContext->bit_rate = 1000000; // 设置视频比特率
    videoCodecContext->width = 1920; // 设置视频宽度
    videoCodecContext->height = 1080; // 设置视频高度
    videoCodecContext->time_base.num = 1;
    videoCodecContext->time_base.den = 30; // 设置帧率
    videoCodecContext->gop_size = 10; // 设置 GOP 大小

    // 设置音频编码器参数
    audioCodecContext = audioStream->codec;
    audioCodecContext->codec_id = AV_CODEC_ID_AAC;
    audioCodecContext->codec_type = AVMEDIA_TYPE_AUDIO;
    audioCodecContext->bit_rate = 64000; // 设置音频比特率
    audioCodecContext->sample_rate = 44100; // 设置音频采样率
    audioCodecContext->channel_layout = AV_CH_LAYOUT_STEREO; // 设置声道布局

    // 打开视频和音频编码器
    if (avcodec_open2(videoCodecContext, avcodec_find_encoder(videoCodecContext->codec_id), NULL) < 0) {
        std::cerr << "Failed to open video encoder." << std::endl;
        return 1;
    }

    if (avcodec_open2(audioCodecContext, avcodec_find_encoder(audioCodecContext->codec_id), NULL) < 0) {
        std::cerr << "Failed to open audio encoder." << std::endl;
        return 1;
    }

    // 打开 RTMP 输出
    if (!(outputFormat->flags & AVFMT_NOFILE)) {
        if (avio_open(&formatContext->pb, formatContext->url, AVIO_FLAG_WRITE) < 0) {
            std::cerr << "Failed to open output URL." << std::endl;
            return 1;
        }
    }

    // 写入文件头
    if (avformat_write_header(formatContext, NULL) < 0) {
        std::cerr << "Failed to write file header." << std::endl;
        return 1;
    }

    // 循环写入视频和音频帧
    AVFrame *videoFrame = av_frame_alloc();
    AVFrame *audioFrame = av_frame_alloc();
    
    // 在这里填充 videoFrame 和 audioFrame 的数据

    while (true) {
        // 将 videoFrame 和 audioFrame 的数据写入流中
        if (videoFrame) {
            videoFrame->pts = av_rescale_q(videoFrame->pts, videoCodecContext->time_base, videoStream->time_base);
            av_interleaved_write_frame(formatContext, videoFrame);
        }
        if (audioFrame) {
            audioFrame->pts = av_rescale_q(audioFrame->pts, audioCodecContext->time_base, audioStream->time_base);
            av_interleaved_write_frame(formatContext, audioFrame);
        }

        // 在这里更新 videoFrame 和 audioFrame 的数据，直到结束

        // 如果结束了，退出循环
        if (/* 检查是否结束的条件 */) {
            break;
        }
    }

    // 写文件尾
    av_write_trailer(formatContext);

    // 清理资源
    avcodec_close(videoCodecContext);
    avcodec_close(audioCodecContext);
    av_frame_free(&videoFrame);
    av_frame_free(&audioFrame);
    avio_close(formatContext->pb);
    avformat_free_context(formatContext);

    return 0;
}
```
##  十一、JT1078协议接入两客一危视频
- 部署jtt808/1078协议组件

JT808/1078协议流媒体服务器程序分为两个版本：一个为流媒体服务器独立运行JT808/1078协议的版本，另一个为流媒体服务器同时支持GB28181协议、JT808/1078协议这两个协议的版本。

进入/opt/JT808-Sever目录，执行“unzip ./JT808-Server.zip”。解压后产生3个文件如下图所示：

![image](https://github.com/user-attachments/assets/bfd26ded-6c86-43e8-b9c8-b5672249db92)

 执行下列命令启动组件：
```
sudo java -jar jtt808-server-1.0.0-SNAPSHOT.jar --spring.config.location=./application.yml
```
当在前台启动成功后，命令行输出信息如下：

![image](https://github.com/user-attachments/assets/64314d7b-5a33-45a8-90a1-2ea402ca556e)

系统默认侦听7611端口，实施人员可以编辑application.yml配置文件，根据生产环境的规划调整侦听端口。配置内容下图所示：

![image](https://github.com/user-attachments/assets/e6dfca90-223e-4140-aac8-454b8fcd3b93)

至此，JT808/1078协议组件部署完毕。

- 部署JT808/1078协议流媒体服务器程序
进入/opt/JT1078-Sever目录，执行“unzip ./JT108-Server.zip”。解压后产生4个文件以及1个lib文件夹，如下图所示：

![image](https://github.com/user-attachments/assets/3c7e9c99-90b6-4f56-bb5c-986ec86ec983)

配置JT808/1078协议流媒体服务器程序lib加载路径。用vim打开~/.bashrc文件，在文件最后一行添加：
```
export LD_LIBRARY_PATH=/opt/JT1078-Server/lib/:$LD_LIBRARY_PATH
```
保存后退出，并执行：
```
source  ~/.bashrc
```
修改推流服务器地址：用vim打开app.properties配置文件，将rtmp.url地址指向MediaServer服务器地址。如下图所示：

![image](https://github.com/user-attachments/assets/83b31767-9905-484d-bc4f-af67ac962cc7)

配置文件app.properties修改完毕后，执行下列命令启动JT1078流媒体服务器主程序（在生产环境中请实施人员在后台启动该服务）：
```
sudo java -jar jtt1078-video-server-1.0-0.jar --spring.config.location=./app.properties
```
至此，JT808/1078流媒体服务器程序部署完毕。

- T808/1078协议协议方式接入海康NVR

本文使用海康威视设备型号为DS-M5504HN。登录此设备管理后台(默认地址：http://192.168.1.66，如果不是这个地址，请咨询设备厂家，默认密码为admin/admin)，进入到配置页面，分别点击[车载]->[两客一危]标签，在展开的页面中选中[平台配置]标签（不同型号的设备配置页面位置可能不相同，请实施人员自行查找，或咨询设备厂家）。需配置项目如下所示：

-- 业务平台选择：选中一个平台即可，本文选择两客一危第一中心（一般情况下有四个中心可以配置，实施人员根据生产环境需求进行配置，如果四个中心都启用，那么车辆注册信息将同时发往四个中心，并在四个中心均可发起车载视频预览等业务）。
-- 启用：勾选，表示启用当前选中平台（本文启用两客一危第一中心业务平台）。
-- 主服务器域名：填写JT808/1078协议组件服务器地址。
-- 主机IP地址：填写JT808/1078协议组件服务器地址。
-- TCP端口：填写JT808/1078协议组件侦听端口，默认情况下端口为7611。
-- UDP端口：填写JT808/1078协议组件侦听端口，默认情况下端口为7611。

![image](https://github.com/user-attachments/assets/877ad576-688b-4698-a3af-beb3bef86e1f)

上述参数配置完毕后，重启设备。设备重启成功后，再次进入到[两客一危]界面，在[注册状态]中可以看到[两客一危第一中心]这个业务平台中[注册状态]为[注册成功]状态。如下图所示：

![image](https://github.com/user-attachments/assets/763107be-0059-4ae2-ab0d-908d01c38f38)

此时，可以预览NVR通道上的实时流媒体数据。具体操作步骤如下：
 
-- 启动postman向JT808/1078协议组件服务器8000端口推送预览请求，请求方法为9101。如下图所示：

![image](https://github.com/user-attachments/assets/e75e504d-9012-4dea-abb8-b0183415a278)

 参数说明如下：
 
1) clientId:填写设备[两客一危]中[注册信息]页面中手机号码。
2) ip：JT808/1078协议流媒体服务器IP地址。
3) tcpPort:JT808/1078协议流媒体服务程序侦听的流媒体端TCP口号。
4) udpPort:JT808/1078协议流媒体服务程序侦听的流媒体端UDP口号。
5) channelNo：终端设备视频通道号。
6) mediaType：数据类型：0=音视频，1=视频。
7) streamType：码流类型：0=主码流，1=子码流

![image](https://github.com/user-attachments/assets/916b5bdd-5e05-441a-a0e0-3164ec7d7831)

-- Postman推送视频预览请求成功后，通过MediaServer的视频工作端口预览视频。本文采用VLC工具进行播放。
MediaServer同时支持GB28181协议和JT808/1078协议时播放路径：
```
http://192.168.1.115:80/live/012107877146-1.live.flv?callId=5837&sign=69df482ab580a0964592d0f0e673a4e7
```
MediaServer独立运行JT808/1078协议时播放路径：
```
http://192.168.1.115:80/live/012107877146-1.live.flv
```
播放画面如下图所示：

![image](https://github.com/user-attachments/assets/9ded65b0-690c-439b-82ca-1c4fc9abaf9c)

同时，可以打开http:192.168.1.115:8000/ws.html订阅终端消息：
![image](https://github.com/user-attachments/assets/9208a60e-14c6-4560-85a2-f6bf262f93aa)

也可以打开http:192.168.1.115:8000/doc.html查看协议接口文档：

![image](https://github.com/user-attachments/assets/425c09f3-3045-443d-9575-707cf4ed1b3f)

至此，JT808/1078协议组件部署完毕

## 十二、wvp的https访问

 如果还没有购买域名，则可以使用自签名证书，如果是自签证书则浏览器端需手动安装自签证书，自签名证书方法参考《OpenSSL自签证书》一文。将证书文件复制到wvp配置文件所在目录。并开启https访问，。如下图所示：
 
 ![image](https://github.com/user-attachments/assets/61d32fc8-6404-4353-8979-468369179268)

再次启动程序后，通过https访问，效果如下：

![image](https://github.com/user-attachments/assets/86832a6e-3ac8-48f1-bf75-ad17001d268b)

 相应的也可以通过https方式请求后台接口。

## 十三、ZLMediaKit开启https访问
ZLMediaKit默认情况下加载自带的default.pem证书。效果如下：

![image](https://github.com/user-attachments/assets/1e4c965f-d137-4767-88e2-d0a891047251)

 如果你的开发机器IP不是证书绑定的域名映射的IP，则可以通过修改host文件来实现测试。以linux/mac为例：
```
#打开host文件
sudo vi /etc/hosts
#新增内容(本机ip+空格+你的域名)
127.0.0.1  test.zlmediakit.com
#修改后保存退出vi
```

 ![image](https://github.com/user-attachments/assets/f39940e0-4e4a-4327-b022-9b343a2b3625)

 当拿到证书后，使用-s 参数加载证书：
```
sudo ./MediaServer -s ./192.168.1.2.pem
```
启动后，可以看到证书正常加载：

![image](https://github.com/user-attachments/assets/c8dbd4ab-21a6-47e9-8c38-14e7146ab0f0)

测试在wvp平台则可以通话https方式预览视频：

![image](https://github.com/user-attachments/assets/e2993c1d-6147-4e70-90fd-3d8812dc2b31)

也可以启用tomcat自建web服务预览画面。在server.xml配置如下：
```
<Connector port="8443" protocol="org.apache.coyote.http11.Http11NioProtocol"
               maxThreads="150" SSLEnabled="true"
               maxParameterCount="1000" scheme="https" secure="true"
               keystoreFile="/home/apache-tomcat-8.5.91/conf/192.168.1.2.p12"
               keystoreType="PKCS12" keystorePass="PKCS12"
               keyAlias="bsoft-media"
                   address="192.168.1.2"
               >
```
v.html内容如下：
```
<script src="./flv.js"></script>
    <video id="videoElement" controls autoplay width="1024" height="576"></video>
    <script>
      if (flvjs.isSupported()) {
        var videoElement = document.getElementById('videoElement');
        var flvPlayer = flvjs.createPlayer({
          type: 'flv',
          url: 'https://192.168.1.2:443/rtp/34020000001110000021_34020000001310000001.live.flv'
        });
        flvPlayer.attachMediaElement(videoElement);
        flvPlayer.load();
        flvPlayer.play();
      }
    </script>
```
浏览器中输入视频预览地址。效果如下图：

![image](https://github.com/user-attachments/assets/6ef0d9d6-e60f-4fa5-95d6-73c384b236e3)

