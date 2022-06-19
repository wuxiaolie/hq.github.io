---
title: VisitorMS - 安卓APP开发
tags: RaspberryPi-project
article_header:
  type: cover
  image:
    src: /20.png
---



### 如何开发一款APP

请参考我的另一篇博文：

Android开发笔记——零基础入门（最终完成一个智能家居APP）（附源码）

https://blog.csdn.net/weixin_45346142/article/details/117922968



### 修改APP名称和页面名称

修改下图选中的文件，直接更改Value值即可。

其中，第四个 `title_activity_welcome` 的Value值关联了APP名称也关联了第一个页面名称。

> ![Snipaste_2022-06-15_09-44-08](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/RaspberryPi/use/Snipaste_2022-06-15_09-44-08.png)
>
> ![Snipaste_2022-06-15_10-01-27](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/RaspberryPi/use/Snipaste_2022-06-15_10-01-27.png)

效果如下

> <img src="https://photo-hq.oss-cn-hangzhou.aliyuncs.com/RaspberryPi/use/Snipaste_2022-06-15_10-01-05.png" alt="Snipaste_2022-06-15_10-01-05" style="zoom:67%;" />
>
> ![Snipaste_2022-06-15_10-01-15](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/RaspberryPi/use/Snipaste_2022-06-15_10-01-15.png)



### 修改APP图标

https://blog.csdn.net/wpwbb510582246/article/details/52556753

通过重新产生工程图标的方法来修改项目的图标，具体如下：

1、点击项目文件夹下的AndroidManifest.xml文件

![img](../../../知识汇总/0 - 个人学习/1 - 工程实践/校招树莓派项目开发/【6】智能家居APP开发/assets/assets.安卓APP开发笔记 - HQ/Center.png)

2、点击Application选项

![img](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/RaspberryPi/use/Center-16552779157061.png)

3、在Icon一行点击Browse，弹出如下对话框

![img](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/RaspberryPi/use/Center-16552779157072.png)

4、点击Creat New Icon，弹出如下对话框

![img](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/RaspberryPi/use/Center-16552779157073.png)

5、然后点击Next弹出配置项目图标的对话框

![img](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/RaspberryPi/use/Center-16552779157074.png)

6、然后再根据自己的需要进行配置项目图标，这样便可以在开发Android项目的过程中来修改项目的图标



### 修改主程序

修改WebView显示网址

> ![image-20220615152724081](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/RaspberryPi/use/image-20220615152724081.png)

修改连接的服务器IP和端口号

> ![image-20220615152842423](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/RaspberryPi/use/image-20220615152842423.png)

修改各个按钮的功能，改变发送的字符串即可

> ![image-20220615152744862](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/RaspberryPi/use/image-20220615152744862.png)



### 修改界面显示

修改背景颜色，background=“xxxxxx” 改为6位颜色编码

> ![image-20220615152931750](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/RaspberryPi/use/image-20220615152931750.png)

修改按钮值

> ![image-20220615153025046](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/RaspberryPi/use/image-20220615153025046.png)

效果如下

> ![image-20220615153244788](https://photo-hq.oss-cn-hangzhou.aliyuncs.com/RaspberryPi/use/image-20220615153244788.png)



































































































































































































































































































