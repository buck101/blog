---
layout:       post
title:        树莓派通过hls实现24*7监控摄像流媒体直播
author:     "Me"
tags:
    - 树莓派
---
<div id='wx_logo' style='margin:0 auto;display:none;'>
<img src='/img/v.jpg'/>
</div>

自从全叔介绍我玩树莓派以来，越来越发现这小东西的可玩性很高。总结起来它可玩性如此高的原因下面几点：
体积小、价格便宜、接口多、能耗少（可做24*7服务器用）、架构通用，软件丰富

这两天买了个原装的[摄像卡][1]（非usb,不能采集音频），搭建一个24*7的监控流媒体播放服务器，步骤如下：

## 1.通过raspberry自带的raspivid采集原始H264流

``` shell
raspivid  -vf -hf -n -ih -t 0 -ISO 800 -ex night -w 720 -h 405 -fps 25 -b 500000 -o -| ffmpeg -y -loglevel panic -i - -c:v copy -map 0 -f ssegment -segment_time 1 -segment_format mpegts -segment_list "/var/www/hls/stream.m3u8" -segment_list_size 80 -segment_wrap 80 -segment_list_flags +live -segment_list_type m3u8 "/var/www/hls/%03d.ts"
```

因为摄像头是倒放的所以加了-vf -hf参数
考虑到联通10M光纤上行只有1M，采集的码率限制500000（500Kbit）

## 2.ffmpeg吃进原始H264流，分隔后吐出hls文件（m3u8和ts）
命令见步骤1
**考虑到长时间对SD卡的不隔断读写，可以使用虚拟内存文件系统**
mount -t tmpfs -o size=16M tmpfs  /var/www/hls
防止吃光内存，最大只分配16M使用。

## 3.前端页面挂上播放地址

``` html
<video width="720" height="480" controls="controls" id="video" autoplay="autoplay" >
  <source src="hls/stream.m3u8" type="application/x-mpegURL"> 
</video>
```
目前只适配VLC/android/IOS的http协议，https对android还不支持.
有时间再研究在PC浏览器支持http/https播放，以及android的https播放

## 4.总结
实际在树莓派2(4核心,1GB RAM)上运行毫无压力，dstat输出如下：
![DSTAT](/img/2015-04-23-raspberry-pi-http-live-streaming/dstat.jpg "dstat")

[1]:http://elinux.org/File:RPiCam.jpg


