---
layout:       post
title:        WebRTC把玩记之一
subtitle:     AppRTC应用环境搭建完整步骤
author:     "Me"
tags:
    - WebRTC
---
<div id='wx_logo' style='margin:0 auto;display:none;'>
<img src='/img/v.jpg'/>
</div>

[![alt text](/img/click.jpg "点我")](https://apprtc.51buck.com)

做了这么久的直播，其实只是高延时的普通直播，和现在市面上大行其道的互动直播区别还是比较大的。互动直播需要更高的实时性和低延时，而且需要适应终端多人连麦的各种需求，这种双向交互的互动直播与电视台式的单向广播式的普通直播在技术上还是有较大的区别。

WebRTC是google在收购Global IP Solutions公司已有技术的基础上开源出来的一套支持网页实时音视频通话框架，国内市面上的互动直播基础上都是在WebRTC的框架上进行的二次开发，很少自己发明轮子的。

利用WebRTC可以衍生出很多好玩的应用，比如[Webcam Toy][1]和[ASCII Camera][2]，值得一提的是我们用了很久的QQ视频聊天其实就是来自Global IP Solutions公司，只不过是WebRTC闭源商业版。下面要介绍的是一个使用WebRTC的典型网站[AppRTC][3]，她能够支持网页上的点对点的音视频聊天，但由于我朝大局域网的原因，这个网站国内是无法访问的。下面我介绍自己搭建一个类AppRTC[网站][4]的步骤。先上图吧

大图是PC上传的流，右下角的小图是手机上采的流。PC走的是普通局域网，手机是走的4G网络。


## 1.依赖环境安装
ubuntu nginx java go node python GAE (google app engine sdk), 这些依赖最好手工下载安装。
nginx安装好后配置一个https的host，我用的域名是**apprtc.51buck.com**，大家参考的时候只要替换成自己的域名就行。

下面是我安装后的bashrc

``` html
export JAVA_HOME=/usr/local/java/jdk1.8.0_151
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$JAVA_HOME/bin:$PATH

export GOROOT=/usr/local/go/v1.9.2
export GOPATH=~/Works/go
export GOBIN=$GOPATH/bin
export GOARCH=amd64
export GOOS=linux
export PATH=$GOROOT/bin:$GOPATH/bin:$PATH

export GAE=/home/buck/Works/Alphabet/google-cloud-sdk
source $GAE/completion.bash.inc
source $GAE/path.bash.inc
export PATH=$GAE/bin:$PATH

export NODE_HOME=/usr/local/node/v8.9.1
export PATH=$NODE_HOME/bin:$PATH

#export http_proxy=http://xxxxx:yyyyy
#export https_proxy=http://xxxxx:yyyyy
```

在安装和初始化GAE的时候要翻墙，记得设置代理服务器。
安装GAE后需要按如下步骤进行初始化，根据提示一步一步做就行：

``` html
cd $GAE
./google-cloud-sdk/install.sh
gcloud init --skip-diagnostics
gcloud components install app-engine-python
gcloud components install app-engine-python-extras
```

## 2.组件安装
一个AppRTC应用需要三个建立在WebRTC框架上的组件
包括**房间服务器**、**信号服务器**、**打孔服务器**。其中前面两个都在apprtc项目中，最后一个在coturn项目中。

### 2.1房间服务器

#### 2.1.1 先安装
``` html
git clone https://github.com/webrtc/apprtc
cd apprtc
npm install -g grunt-cli
npm install
```

#### 2.1.2 再修改两处源文件：
src/app_engine/constants.py

``` python
TURN_BASE_URL = 'https://computeengineondemand.appspot.com'
改成
TURN_BASE_URL = 'https://apprtc.51buck.com'

ICE_SERVER_BASE_URL = 'https://networktraversal.googleapis.com' 
改成
ICE_SERVER_BASE_URL = 'https://apprtc.51buck.com'

两处
WSS_INSTANCE_HOST_KEY: 'apprtc-ws.webrtc.org:443'
改成
WSS_INSTANCE_HOST_KEY: 'apprtc.51buck.com:28089'
```

./src/web_app/js/appcontroller.js
添加一行代码强制将https转成http：

``` js
#Add By Buck
roomLink=roomLink.substring("http","https");
window.history.pushState({"roomId":roomId, "roomLink":roomLink}, roomId, roomLink);
```

#### 2.1.3 再编译：
``` shell
grunt build
```

#### 2.1.4 后台启动即可

``` shell
nohup dev_appserver.py --host=127.0.0.1 --port 18085 --admin_host=0.0.0.0 --admin_port 18000 --log_level debug out/app_engine/ &
```

### 2.2信号服务器
#### 2.2.1 在apprtc项目目录执行
``` html
 ln -s `pwd`/src/collider/collider $GOPATH/src
 ln -s `pwd`/src/collider/collidermain $GOPATH/src
 ln -s `pwd`/src/collider/collidertest $GOPATH/src
 cd $GOPATH/src/collidermain
```
#### 2.2.2 修改文件main.go:
``` go
将
var roomSrv = flag.String("room-server", "https://appr.tc", "The origin of the room server")
改成
var roomSrv = flag.String("room-server", "https://apprtc.51buck.com", "The origin of the room server")
```
#### 2.2.3 生成验证文件：
``` html
mkdir /cert
openssl genrsa -des3 -out key.pem 2048
openssl req -new -key key.pem -out server.csr
cp key.pem server.key.org
openssl rsa -in server.key.org -out key.pem
openssl x509 -req -days 365 -in server.csr -signkey key.pem -out cert.pem
```

#### 2.2.4 后台运行服务
``` html
nohup ./collidermain -port=28089 -tls=true -room-server='http://127.0.0.1:18085' &
```

### 2.3打孔服务器

#### 2.3.1 安装

``` html
git clone https://github.com/coturn/coturn
cd coturn
./configure
make -j 4
make install
```

#### 2.3.2 添加用户

``` html
  turnadmin -a -u buck -r apprtc.51buck.com -p buck
  turnadmin -A -u buck -p buck
  turnadmin -k -u buck -p buck -r apprtc.51buck.com
  0xe00a389a1f3dad029491a16c254e32b5
记住上面这条输出密码，填入下面的配置文件中
```

#### 2.3.3 修改配置

examples/etc/turnserver.conf
``` html
listening-device=eth0
listening-port=3478
relay-device=eth0
min-port=59000
max-port=60000
Verbose
fingerprint
lt-cred-mech
use-auth-secret
static-auth-secret=buck
user=buck:buck
user=buck:0x9db00ade3d323a3f9d1180b72983e5e9
realm=apprtc.51buck.com
stale-nonce=600
cert=/root/coturn/examples/etc/turn_server_cert.pem
pkey=/root/coturn/examples/etc/turn_server_pkey.pem
no-loopback-peers
no-multicast-peers
mobility
no-cli
```

#### 2.3.3 后台启动服务：

``` shell
nohup ./bin/turnserver -a -c examples/etc/turnserver.conf &
```

上面具体参数的含义这就不多解释了，自行google吧

### 2.4其它辅助服务器

#### 2.4.1 nginx
last but not least, 还需要配置nginx服务器将用户的url请求转发到对应的上述三个服务器上
如何配置https这里就不说了，在这基础上添加两项路由匹配

``` html
 location / {
      #root   html;
      #index  testssl.html index.html index.htm;

      proxy_redirect off;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_pass http://127.0.0.1:18085/;
  }

  location /v1alpha/iceconfig { 
      #add_header Access-Control-Allow-Origin *;
      #proxy_redirect off;
      #proxy_set_header Host $host;
      #proxy_set_header X-Real-IP $remote_addr;
      #proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_pass http://127.0.0.1:3033/v1alpha/iceconfig; 
  } 
```
#### 2.4.2 打孔查询服务器
上面的两条路由，第一条是转发到房间服务器的，第二条是为了获取打孔服务器列表的，返回列表的服务器用js简单写一个就行
主要逻辑就是监听3033端口，处理转发过来的http post请求，返回2.3中设置好的打孔服务器列表。

文件名ice.js
``` js
var express = require('express')
var crypto = require('crypto')
var app = express()

var hmac = function (key, content) {
  var method = crypto.createHmac('sha1', key)
  method.setEncoding('base64')
  method.write(content)
  method.end()
  return method.read()
}

app.post('/v1alpha/iceconfig', function (req, resp) {
  var query = req.query
  var key = 'apprtc.51buck.com'
  var time_to_live = 600
  var timestamp = Math.floor(Date.now() / 1000) + time_to_live
  var turn_username = timestamp + ':buck'
  var password = hmac(key, turn_username)

  return resp.send({
    iceServers: [
      {
        urls: [
          'stun:apprtc.51buck.com:3478',
          'turn:apprtc.51buck.com:3478'
        ],
        username: turn_username,
        credential: password
      }
    ]
  })
})

app.listen('3033', function () {
  console.log('server started')
})
```
后台启动
``` shell
nohup node ice.js &
```

## 3.结语

搭好后实现测试发现，PC Chrome和Android使用都没啥问题，但iOS无法使用，记得今年6月份iOS 11已经全面支持WebRTC，具体是啥原因后面再分析吧。
搭好环境只是第一步，接下来需要进一步研究WebRTC的编译和源码实现了，包括研究其如何用UDP保证低延时传输音视频流，如何处理网络抖动保证传输质量等。

[1]:https://webcamtoy.com/zh/
[2]:https://idevelop.ro/ascii-camera/
[3]:https://appr.tc
[4]:https://apprtc.51buck.com

