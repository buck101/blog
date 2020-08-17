---
layout:       post
title:        树莓派监控室内pm2.5数据并使用grafana+influxdb展现
author:     "Me"
tags:
    - 树莓派
---
<div id='wx_logo' style='margin:0 auto;display:none;'>
<img src='/img/v.jpg'/>
</div>

2016快要结束的一个多月,北京的雾霾天又开始严重起来了,而且这一次生理上的不适感比之前大了不少。
也许是经过北京这几年生活累加伤害所至吧。

调研了一下pm2.5对人体的危害和中美空气污染的标准，感觉事情比我原来想的要严重很多。
考虑到家里有老人和小孩，于是决定对此事认真起来，希望找出一个彻底的解决方案。

在定解决方案之前，先来个定量分析吧。可是网上卖的测量仪基本都要小一千（比如汉王的霾表）而且这种产品还是不方便动态监测pm2.5的变化。


最后在淘宝上看到有人拿攀藤科技的传感器做了[简易测量仪][1]，而且还能提供数据的输出接口。

下面记录了使用的步骤和分析得到的结论

## 1.获取室内可吸入颗粒物pm2.5/pm10的浓度

在定制的攀藤传感器上接一个usb模，连接到pi上，通过下面的python代码可以读取数据，并且根据传感器文档，能够解析出具体的数据

``` ruby
ser = serial.Serial(port='/dev/ttyUSB0', timeout=1)
data_n = 37
read_n = 37
head = "\x42\x4d"
pool = ""
while True:
    x = ser.read(read_n)
    pool += x
    offset = pool.find(head)
    if offset == 0:
        break
    elif offset > 0:
        pool = pool[offset:]
        if len(pool) == data_n:
            break;
        else:
            read_n = data_n - len(pool)
if len(pool) == data_n:
    frame = struct.unpack(">HHHHHHHHHHHHHHHHHHc", pool)
```

## 2.获取室外污染指数

使用的是aqicn.org上的数据，该网站提供的API用于在程序中实时获取当前数据，只需要注册一个用户就行。
xxx是你注册用户后得到的一个私有key

``` ruby
def get_aqicn():
	weather_url = "http://api.waqi.info/feed/beijing/?token=xxx"
    req = urllib2.Request(weather_url, headers=headers)
    resp = urllib2.urlopen(req)
    content = resp.read()
    if(content):
        weatherJSON = json.JSONDecoder().decode(content)
        try:
            if weatherJSON['status'] == "ok":
                if weatherJSON['data'].has_key('iaqi'):
                    a =  int(weatherJSON['data']['iaqi']['pm25']['v'])
                    b =  int(weatherJSON['data']['iaqi']['pm10']['v'])
                    return (a, b)
                else:
                    print "no iaqi found"
                    return None
            else:
                print "no ok found"
                return None
        except:
            print "except found"
            return None
```
**由于aqicn.org只提供污染指数而没有提供具体的污染物浓度，为了便与和室内的数据进行对比，可以将室内的污染物浓度换算成污染指数。**
具体换算公式可以在[果壳网][2]或wikipedia上得到

## 3.将采集到的数据进行图形化展示

图形化方案使用的是**telegraf(数据收集)+influxdb(数据存储)+grafana(数据展示)**，具体的安装配置下次有空再说了

``` ruby
msg = "aqi,loc=%s pm25=%d,pm25_aqi=%d,pm10=%d,pm10_aqi=%d,form=%f %ld" % \
                ("outdoor", 0, outdoor[0], 0, outdoor[1], 0, ts)
```
这里就说一点，采集到的数据只要按上述格式往telegraf的监听地址发送即可写入influxdb中。
这里可以将influxdb和普通的关系型数据库mysql做个对比
aqi相当于mysql的一张表（可自动生成）
loc相当于字段（主键）
pm25,pm25_aqi,pm10,pm10_aqi,form等相当于非主键（注意loc和pm25之间是空格分隔而不是逗号）
最后的%ld ts是influxdb特有的时间戳数据，因为influxdb就是用来存储时间序列值的
至于上述数据是写到哪个库中，是在telegraf的配置中指定，每个telegraf进程实例接收到的数据都会全部写到指定的数据库中

## 4.结论
- 在关窗的情况下，室内的污染指数相当于室外的一半。也就是说如果室外是300，室内就是150.这在美标的显示中是**“very unhealthy”**
所以不要以为晚上你的卧室的空气很好，其实在睡眠的时候你是一直在受到伤害的。

- 自住房内空气净化的最佳解决方案应该是新风系统，不但能净化空气而且能保持室内充足的含氧量。
次优的解决方案是[ffu][3]，它的净化效果（CARD）是普通净化器的3-5倍，价格也很亲民，而且强风档噪音只相当于普通的立式空调。
最终我采用的是ffu+普通净化器的方案，客厅放ffu每个卧室放小米净化器。
基本上能在室外爆表的情况下，室内pm2.5浓度不超过40毫克/立方米，基本接近美标优秀（20）的水平。

- 对于办公室的净化方案，可以用[肺宝][4]，虽然有点不美观，但效果是很好的。

- 口说无凭，数据说话，下面列出的2017年元旦北京最长的一次跨年雾霾期间室内、室外、办公室的污染对比
**(家中pm2.5浓度的波峰是因为做饭油烟所致与室外污染物无关)**

![AQI](/img/2016-12-29-raspberry-pi-pm25-monitor/aqi.jpg "aqi")

[1]:https://item.taobao.com/item.htm?spm=a1z09.2.0.0.0G9oyg&id=39422337670&_u=i40493rf6c2
[2]:http://www.guokr.com/post/431588/
[3]:https://detail.tmall.com/item.htm?id=540103289206&spm=a1z09.2.0.0.0G9oyg&_u=i40493r89cd
[4]:https://item.jd.com/1753326931.html


