---
title: 树莓派开发 - MJPG-Streamer视频框架
tags: RaspberryPi-use
article_header:
  type: cover
  image:
    src: https://photo-hq.oss-cn-hangzhou.aliyuncs.com/cover/12.png
---



## MJPG-Streamer 

### MJPG-Streamer 是什么？

简单地说，Mjpg-Streamer 是一个 JPEG 文件的传输流。

它最常用的**用途就是采集摄像头的数据，然后启动 http server，用户就可以通过浏览器查看图像数据了。**

类似 Linux 下的管道：

```
$ cat /dev/videoX | encode to JPG | http_server 
```

**官网**：

> https://sourceforge.net/projects/mjpg-streamer/

**Github**：

> https://github.com/jacksonliam/mjpg-streamer

**什么是 MJPG？**

Motion JPEG，简称 MJPG。

JPEG是静态图片的编码格式。

MJPG 是动态的视频编码格式，可以简单地理解：MJPG 就是把多个 JPEG 图片连续显示出来。

**MJPG 的优点？**

很多摄像头本身就支持 JPEG、MJPG，所以处理器不需要做太多处理。

一般的低性能处理器就可以传输 MJPG 视频流。

**MJPG 的缺点？**

MJPG 只是多个 JPEG 图片的组合，它不考虑前后两帧数据的变化，总是传输一帧完整的图像，传输带宽要求高。

H264 等视频格式，会考虑前后两帧数据的变化，只传输变化的数据，传输带宽要求低。

**个人感受：**

这个项目虽然已经多年不更新了，但是却一直有人使用，说明其代码品质较好。

符合 UNIX 的设计哲学：简单实用。

同时，项目的整体设计清晰，扩展性好，可读性高，**非常适合用来训练 Linux 系统下的网络和多线程编程。**





### MJPG-Streamer 怎么用？

MJPG-Streamer的依赖很少。

**编译很简单**：

```
$ sudo apt-get install cmake libjpeg8-dev
$ make
```

**最常见的用法**：

```
$ export LD_LIBRARY_PATH=`pwd`
$ ./mjpg_streamer -i "input_uvc.so -d /dev/video2 -r 640x480 -f 30" -o "output_http.so -w www"
 i: Using V4L2 device.: /dev/video0
 i: Desired Resolution: 1920 x 1080
 i: Frames Per Second.: 30
 i: Format............: JPEG
 o: www-folder-path......: ./www/
 o: HTTP TCP port........: 8080
```

MJPG-Streamer 将图像的来源视为输入，将图像的展示视为输出。

**每一种输入或输出是都抽象为一个插件。**

上面的例子中：

- -i 指定输入的插件为 input_uvc.so，此插件会从摄像头中采集图像。

- 子参数 -d 指定设备节点、-r 指定分辨率、-f 指定 fps。


- -o 指定输出的插件为 output_http.so，此插件会启动一个 http server。

MJPG-Streamer 启动后，在浏览器中输入：

```
127.0.0.1:8080
```

就可以看到图像了。





###  MJPG-Streamer 怎么实现的？

想要理解 MJPG-Streamer 的核心设计，要抓住 2个要点：

**1、工作模型**。

> ![图片](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/use640-16501669121321.png)

MJPG-Streamer 工作时，除了 main 线程，还会起 2 个线程。

输入插件的线程 A 负责从摄像头采集一帧数据，暂存在 buffer 中。

输出插件的线程 B 负责实现 http server，当有 http client 要求访问时图像时，从 buffer 中读一帧数据，然后发送给 http client。

**2、如何实现插件化**。

MJPG-Streamer 将每一种输入或输出是都抽象为一个插件。

每一个插件，都必须实现 3 个 API:

```c
int (*init)(output_parameter *param, int id);

int (*run)(int);

int (*stop)(int);
```

- init() 负责初始化插件;


- stop() 负责让插件停止工作;


- run() 负责让插件开始工作;


每一个插件，最终都会被编译成一个动态库。

- main 线程中通过 dlopen、dlsym() 对插件进行统一地调度使用。


**核心数据结构：**

1、对输入插件的抽象

```c
struct _input {
    char *plugin;
    
    [...]

    // 互斥访问
    pthread_mutex_t db;
    pthread_cond_t db_update;

    // 存储图像数据
    unsigned char *buf;
    int size;

    // 统一的 API
    int (*init)(input_parameter *, int id);
    int (*stop)(int);
    int (*run)(int);
};
```

2、对输出插件的抽象

```c
struct _output {
    [...]

    // 统一的 API
    int (*init)(output_parameter *param, int id);
    int (*stop)(int);
    int (*run)(int);
};
```

这两个数据结构基本就确定了 MPJG-Streamer的核心框架。

如果你想了解 input_uvc.so 这个插件是如何采集摄像头数据的，那么只需要阅读其如何实现 init()、stop()、run() 这几个 API 即可。

如果你想给 MJPG-Streamer 增加功能，例如你想让其支持使用 live555 进行流媒体传输，那么你需要先学会使用 live555，然后将其用法封装成 init()、stop()、run() 供 MJPG-Streamer 调用即可。

### 总结

MJPG-Streamer 虽然老旧，但是其设计理念遵循了 UNIX 的设计哲学，Keep it simple。





## 树莓派摄像头配置流程

> [摘自文章](https://jingyan.baidu.com/article/47a29f2474a555c01523994c.html)

树莓派利用pi Camera模块，通过mjpg-streamer软件获取视频，通过手机端或电脑端浏览实时视频。

### 步骤1

1. sudo apt-get update  #更新软件列表

   sudo apt-get upgrade #更新软件

2. sudo apt-get install subversion #Subversion是一个自由开源的版本控制系统

3. sudo apt-get install libjpeg8-dev #JPEG支持库

   sudo apt-get install imagemagick

   sudo apt-get install libv4l-dev  #4l是小写"L"

   sudo apt-get install cmake #下载编译工具


### 步骤2

1. sudo apt-get install git

2. git clone https://github.com/jacksonliam/mjpg-streamer.git

   > <img src="https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/usea44e8afc508c9bced7e680c4d6dd884ce44afab0.jpg" alt="树莓派3B + Pi摄像头+mjpg-streamer" style="zoom:67%;" />

3. cd mjpg-streamer/mjpg-streamer-experimental #进入下载目录后进入左侧路径


### 步骤3

1. make all #编译

   > <img src="https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/use890dfb4a2f27e7ef3f1861b219dd3340b7f3f5b0.jpg" alt="树莓派3B + Pi摄像头+mjpg-streamer" style="zoom:67%;" />

2. sudo make install #安装

   > <img src="https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/use2e66f9ef28066b018bc339f33df39187021cf3b0.jpg" alt="树莓派3B + Pi摄像头+mjpg-streamer" style="zoom: 67%;" />

3. 修改树莓派系统设置，打开摄像头，重启树莓派

   `sudo raspi-config`

   > ![image-20220514111001939](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useimage-20220514111001939.png)
   >
   > ![image-20220514111018129](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useimage-20220514111018129.png)

4. 修改启动脚本 `vi start.sh` 

   > ![image-20220514111111683](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useimage-20220514111111683.png)

5. 运行 `./start.sh` 此处因为没连接摄像头，提示失败

   >![image-20220514111440602](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useimage-20220514111440602.png)

   其他参考方法

   > `sudo mjpg_streamer -i "./input_uvc.so -r 640x480 -f 10 -n" -o "./output_http.so -p 8080 -w /usr/local/www" `
   >
   > 此命令尤为重要，如下图所示，输出信息，说明成功！
   >
   > <img src="https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useb7b28f87031c99c0b0e2eb35af2fa872951fedb0.jpg" alt="树莓派3B + Pi摄像头+mjpg-streamer" style="zoom:67%;" />

6. 在浏览器输入 `http://IP地址:8080`，回车 显示如下页面，点击页面左侧，Stream栏，显示监视画面

   > <img src="https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useaebdff86242fa872b701b457bfdaf05e4b23e9b0.jpg" alt="树莓派3B + Pi摄像头+mjpg-streamer" style="zoom:67%;" />





## 【树莓派】网络视频监控 - 编程那些年

> https://mp.weixin.qq.com/s/4-pMaQdpekXrVC7QFXE2Pw

利用树莓派和CSI摄像头，通过两种常见方案，我们可以简单实现局域网内的实时视频监控，接下来就讲解下如何部署这两种方案。

### 测试环境

**硬件：**树莓派3B/3B+
**系统：**Raspberry Debian 9 / Debian 10

### 准备工作

首先，按《[【树莓派】让你的SD卡快速扩容](http://mp.weixin.qq.com/s?__biz=MzkzMDE4MDM2NQ==&mid=2247483971&idx=1&sn=518b9e1bf8dce24754eb3cc00bd5ed55&chksm=c27f7e21f508f737583a9f06be5717ea241164806b70168b823d96fef2e4b2f98413c3427289&scene=21#wechat_redirect)》方法对SD卡进行扩容，防止空间不足的问题
其次，根据《[【树莓派】CSI摄像头简单配置](http://mp.weixin.qq.com/s?__biz=MzkzMDE4MDM2NQ==&mid=2247484041&idx=1&sn=2e128e6df8726622a9eb11e65de66813&chksm=c27f7eebf508f7fdb293193eb8ed27231c488fa6ac8d88a3a67fbd603482ce0516483a565351&scene=21#wechat_redirect)》正确连接CSI摄像头，并保证能够正常拍照

### 方案一、利用mjpg-streamer框架实现

更新下树莓派的软件源

```
$ sudo apt update 
```

安装编译mjpg-streamer所需的依赖包

```
$ sudo apt install cmake libjpeg8-dev libv4l-dev   #libv4l是小写"L"
```

下载mjpg-streamer源码包并对其解压

```
$ wget https://github.com/Five-great/mjpg-streamer/archive/master.zip
$ unzip  master.zip 
$ cd mjpg-streamer-master/mjpg-streamer-experimental/
```

或者通过git命令直接下载最新的源码

```
$ git clone https://github.com/jacksonliam/mjpg-streamer.git --depth=1
$ cd mjpg-streamer/mjpg-streamer-experimental 
```

对源码进行编译并安装

```
$ make
$ sudo make install
```

对树莓派的摄像头节点进行确认

```
pi@raspberrypi:~/mjpg-streamer-master/mjpg-streamer-experimental $ ls /dev/video*
/dev/video0  /dev/video10  /dev/video11  /dev/video12
```

如果存在video**(\*为数字,如video0)*的设备节点，说明可以走uvc通道，直接运行start.sh脚步即可。

```
pi@raspberrypi:~/mjpg-streamer-master/mjpg-streamer-experimental $ chmod +x start.sh
pi@raspberrypi:~/mjpg-streamer-master/mjpg-streamer-experimental $ ./start.sh 
MJPG Streamer Version: git rev: 4be5902ba243d597f8d642da5b5271f25e2fb44d
 i: Using V4L2 device.: /dev/video0 #设备节点就是/dev/vide0
 i: Desired Resolution: 640 x 480
 i: Frames Per Second.: -1
 i: Format............: JPEG
 i: TV-Norm...........: DEFAULT
省略。。。。
 o: www-folder-path......: ./www/
 o: HTTP TCP port........: 8080
 o: HTTP Listen Address..: (null)
 o: username:password....: disabled
 o: commands.............: enabled
```

如果不存在video* *(***为数字，如video0)*的设备节点，则需要修改start.sh脚步，将input_uvc.so修改为input_raspicam.so， 如下：

```
pi@raspberrypi:~/mjpg-streamer-master/mjpg-streamer-experimental $  vi  startup.sh
export LD_LIBRARY_PATH="$(pwd)"
#./mjpg_streamer -i "input_uvc.so --help"

#./mjpg_streamer -i "./input_uvc.so" -o "./output_http.so -w ./www"
./mjpg_streamer -i "./input_raspicam.so" -o "./output_http.so -w ./www" #增加该字段
#./mjpg_streamer -i "./input_uvc.so -n -f 30 -r 1280x960"  -o "./output_http.so -w ./www"
#./mjpg_streamer -i "./input_uvc.so -n -f 30 -r 640x480 -d /dev/video0"  -o "./output_http.so -w ./www" &
#./mjpg_streamer -i "./input_uvc.so -d /dev/video0" -i "./input_uvc.so -d /dev/video1" -o "./output_http.so -w ./www"
#valgrind ./mjpg_streamer -i "./input_uvc.so" -o "./output_http.so -w ./www"
```

然后运行start.sh脚步即可。

```
pi@raspberrypi:~/mjpg-streamer-master/mjpg-streamer-experimental $ chmod +x start.sh
pi@raspberrypi:~/mjpg-streamer-master/mjpg-streamer-experimental $ ./start.sh 
MJPG Streamer Version.: 2.0
 i: fps.............: 5
 i: resolution........: 640 x 480
 i: camera parameters..............:

Sharpness 0, Contrast 0, Brightness 50
Saturation 0, ISO 0, Video Stabilisation No, Exposure compensation 0
Exposure Mode 'auto', AWB Mode 'auto', Image Effect 'none'
Metering Mode 'average', Colour Effect Enabled No with U = 128, V = 128
Rotation 0, hflip No, vflip No
ROI x 0.000000, y 0.000000, w 1.000000 h 1.000000
 o: www-folder-path......: ./www/
 o: HTTP TCP port........: 8080
 o: HTTP Listen Address..: (null)
 o: username:password....: disabled
 o: commands.............: enabled
 i: Starting Camera
Encoder Buffer Size 81920
```

然后打开浏览器，网址输入http://192.168.1.107:8080，即可看到监控视频效果:*（PS：192.168.1.107为树莓派IP，可通过ifconfig命令确认实际IP）*

> ![image-20220514110638232](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useimage-20220514110638232.png)

此时，在树莓派上，可通过wget命令对视频进行截图，如下：

```
$ wget  http://192.168.1.107:8080/?action=snapshot -O ./image1.jpg 
#这其中192.168.1.107为实际的ip，image1.jpg为要保存到的图片名称
```

**最后，可以创建并添加开机启动的配置文件，实现开机自动启动视频监控:**

创建要使用的开机脚本

```
pi@raspberrypi:~ $ cd 
pi@raspberrypi:~ $ vi test.sh  #/home/pi/test.sh路径

#!/bin/bash

cd  /home/pi/mjpg-streamer/mjpg-streamer-experimental/
./start.sh
```

添加配置文件

```
pi@raspberrypi:/etc/xdg/autostart $ cd   /etc/xdg/autostart/
pi@raspberrypi:/etc/xdg/autostart $ sudo cp xcompmgr.desktop  mjpg.desktop 
pi@raspberrypi:/etc/xdg/autostart $sudo  vim.tiny  mjpg.desktop   

[Desktop Entry]
Type=Application
Name=mjpeg-http
Comment=Start mjpeg-http compositor
NoDisplay=true
Exec=/home/pi/test.sh

保存退出，并重启树莓派。服务就会自动起来
```

### 方案二、利用VLC串流实时输出网络视频流

首先安装VLC软件包

```
$ sudo apt update
$ sudo apt install vlc # Raspberry Debian 10 可以不装
```

然后运行如下命令：

```
pi@raspberrypi:~ $ sudo raspivid -o - -t 0 -w 640 -h 360 -fps 25|cvlc -vvv stream:///dev/stdin --sout '#standard{access=http,mux=ts,dst=:8090}' :demux=h264 

VLC media player 3.0.8 Vetinari (revision 3.0.8-0-gf350b6b5a7)
[009b5b58] main libvlc debug: VLC media player - 3.0.8 Vetinari
[009b5b58] main libvlc debug: Copyright © 1996-2019 the VideoLAN team
[009b5b58] main libvlc debug: revision 3.0.8-0-gf350b6b5a7
省略...
[73600668] main input debug: Buffering 43%
```

即可开启raspivid视频捕捉并发送到VLC。

在PC端安装VLC软件,下载地址如下：
*(PS:偷懒的也可以在公众号后台回复"VLC"获取64位版本)*

```
https://www.videolan.org/
```

安装时，一路默认即可。

最后打开安装好的VLC软件，选择**"媒体（M）->流（S）…->网络（N）"**，输入

```
http://192.168.1.107:8090 #192.168.1.107为树莓派实际IP
```

如下图所示：![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

> ![图片](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/use640-16519971292811.png)
>
> ![图片](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/use640-16519971292822.png)
>
> ![image-20220508160755732](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/useimage-20220508160755732.png)
>
> ![图片](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/use640-16519971292823.png)
>
> ![图片](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/use640-16519971292824.png)
>
> ![图片](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/use640-16519971292825.png)

等待一会即可播出树莓派捕捉的视频流。





## 强大无比的嵌入式多媒体开发神器：SDL2

### SDL 是什么？

==SDL（Simple DirectMedia Layer）是一套开源的跨平台多媒体开发库，使用 C 语言写成。==

它提供了绘制图像、播放声音、获取键盘输入等相关的 API，大大降低多媒体应用开发难度的同时，也让开发者只要用相同或是相似的代码就可以开发出跨多个平台（Linux、Windows、Mac OS X等）的应用软件。

多用于开发游戏、模拟器、媒体播放器等多媒体应用领域。

SDL 有两个常见版本：SDL1.2 和 SDL2.x。在不支持 OpenGL ES2 的嵌入式平台上，只能使用 SDL1.2，SDL2.x 依赖 OpengGL ES2。

**官网**：

> https://sourceforge.net/projects/mjpg-streamer/

**Github**：

> https://github.com/libsdl-org/SDL



### 示例：显示图片

```c
 void main()
 {
    SDL_Window *window = SDL_CreateWindow("Hello World!", 100, 100, 640, 480, SDL_WINDOW_SHOWN);
    
    /* Create a Render */
    SDL_Renderer *render = SDL_CreateRenderer(window, -1, SDL_RENDERER_ACCELERATED | SDL_RENDERER_PRESENTVSYNC);
    
    /* Load bitmap image */
    SDL_Surface *bmp = SDL_LoadBMP("./hello.bmp");
    
    SDL_Texture *texture = SDL_CreateTextureFromSurface(render, bmp);
    SDL_FreeSurface(bmp);

    SDL_RenderClear(render);
    SDL_RenderCopy(render, texture, NULL, NULL); // Copy the texture into render
    SDL_RenderPresent(render); // Show render on window

    /* Free all objects*/
    [...]

    /* Quit program */
    SDL_Quit();
    return 0;
}
```

**要快速上手 SDL，需要理解下面几个核心结构体和 API：**

对于这些结构体，网上有各种各样的解释，但是都比较晦涩生硬，下面是我理解。

**SDL_Window**

想要显示东西，首先要有一个容器来容纳程序绘制的内容。

这个容器，最常见的就是窗口。

对于 PC 机，一般都会有窗口的支持。

但是即便是没有桌面环境的嵌入式平台，SDL_Window 也能兼容，只不过这个窗口没有最小化、最大化、拉伸的功能罢了，都是纯软件的概念，和硬件无关。

一般只需要关注 x、y、w、h 4个成员变量，即窗口的原点坐标和宽高。

通过 SDL_CreateWindow() 可以创建一个窗口。

**SDL_Renderer**

当描述用 GPU 来进行绘画图像时，就用术语渲染。

渲染器，你可以认为就是画笔。

现实世界里，你可以用圆珠笔来画画，也可以用铅笔来画画，所以渲染器也有多种类型，不同的渲染器就好比不同的笔。

SDL2 支持的渲染器：

> <img src="https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/use640-16501670112644.png" alt="图片" style="zoom:50%;" />

==在嵌入式平台上，一般就只支持 OpengGL ES2 这种渲染器类型。==

通过 SDL_CreateRenderer() 可以创建一个渲染器。

**SDL_Texture**

渲染器有了之后，就轮到要渲染的内容了。

Texture 负责描述渲染的内容。

既然是要显示图像，自然需要像素数据、像素格式、宽度、高度、渲染器类型等信息，这些都包含在 Texture 里。

最常见的情况：
调用SDL_CreateTextureFromSurface() 基于某个 Surface 来创建 Texture。Surface 用于描述像素数据。

**SDL_RenderCopy()/SDL_RenderPresent()**

我们把内存想象成是一块备用的画布，而此时屏幕正在显示另一块画布的内容。

你可以不断地调用 SDL_RenderCopy() 来往备用画布上画东西，无论你画多少次都可以。

只有当你调用 SDL_RenderPresent() 时，备用画布才会真正第切换到屏幕上。



###  实践应用

MJPG-Streamer 将每一种输入或输出是都抽象为一个插件。

例如采集摄像头的功能被实现为一个输入插件， 而 http 视频流被实现为一个输出插件。

==我们可以为 MJPG-Streamer 添加一个 SDL2 输出插件。==

==这样就可以在本地显示屏上查看到浏览器的图像了==，效果如下：

```
$ ./mjpg_streamer -i "input_uvc.so /dev/video0 -r 640x480 -f 30" -o "output_sdl2.so"
```

代码很简单，核心内容就是使用 SDL2 显示 JPG 图片，代码就不贴了。



###  内部实现

最后，我想简单地看一下 SDL2 的内部实现。

以最重要的显示功能为例。

SDL2 将底层显示封装为 video driver：

> <img src="https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/use640-16501670112656.png" alt="图片" style="zoom:50%;" />

我们比较熟悉的包括：Wayland、X11、KMSDRM。

以 KMSDRM 这个 video driver 为例，为了支持这个显示驱动，需要实现这么多 API：

> <img src="https://photo-hq.oss-cn-hangzhou.aliyuncs.com/Raspberry/use640-16501670112657.png" alt="图片" style="zoom:50%;" />

如果你想了解 SDL2 是如何实现屏幕显示的，可以尝试阅读这些 API。

### 总结

SDL2 的功能非常丰富，代码质量也很高。

==如果你想在板子进行多媒体开发，例如显示一些东西，又不想用 Qt 那么庞大的图形框架，可以考虑基于 SDL2 进行开发。==











































































































































































































































































