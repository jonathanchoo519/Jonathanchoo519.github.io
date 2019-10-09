---
layout:     post
title:      2CentOS使用ShadowSocks搭配Privoxy上网的方法
date:       2019-10-9
author:     JC
header-img: img/centos.jpg
catalog: false
tags:
    - CentOS
    -ShadowSocks
----

## 1.安装客户端shadowsocks
```
$ yum install python-pip
$ pip install shadowsocks
Pip 是 Python 的包管理工具，这里我们用 pip 安装 shadowsocks。
```
## 2.编辑配置文件

```
$ mkdir -p /etc/shadowsocks
$ vim /etc/shadowsocks/shadowsocks.json

添加以下内容
{ 
    "server":"XXXXX",
    "server_port":1300,
    "local_address": "127.0.0.1",
    "local_port":1080,
    "password":"123456",
    "timeout":300,
    "method":"aes-256-cfb",
    "fast_open": false,
    "workers": 1
}

参数说明：
server：Shadowsocks服务器地址
server_port：Shadowsocks服务器端口
local_address：本地IP
local_port：本地端口
password：Shadowsocks连接密码
timeout：等待超时时间

method：加密方式
可选aes-128-cfb, aes-192-cfb, aes-256-cfb, bf-cfb, cast5-cfb, des-cfb, rc4-md5, chacha20, salsa20, rc4, table

workers:工作线程数
fast_open：true或false。开启fast_open以降低延迟，但要求Linux内核在3.7+。
开启方法 echo 3 > /proc/sys/net/ipv4/tcp_fastopen
```

## 3.配置自启动

新建启动脚本/usr/lib/systemd/system/shadowsocks,内容如下

```
[Unit]
Description=Shadowsocks 
[Service] 
TimeoutStartSec=0 
ExecStart=/usr/bin/sslocal -c /etc/shadowsocks/shadowsocks.json 
[Install] WantedBy=multi-user.target

使用脚本启动
systemctl enable shadowsocks
systemctl start shadowsocks
systemctl status shadowsocks


也可以s执行命令
$ nohup sslocal -c /etc/shadowsocks.json  /dev/null 2>&1 &
```

## 4.验证Shadowsocks客户端是否正常运行
```
curl --socks5 127.0.0.1:1080 http://httpbin.org/ip
#这里的配置地址就是我们上面配置客户端的IP和端口

若Shadowsock客户端已正常运行，则结果如下：
 {
      "origin": "x.x.x.x"       #你的Shadowsock服务器IP
}
```

## 5. 安装配置Privoxy

Shadowsocks是一个 socket5 服务，我们需要使用 Privoxy 把流量转到 http／https 上。

虽然socks代理起来了，但是还不能直接用，需要privoxy转发一下

```
安装Privoxy
$ yum -y install privoxy

启动Privoxy
$ systemctl enable privoxy
$ systemctl start privoxy
$ systemctl status privoxy

配置Privoxy
修改配置文件/etc/privoxy/config

$ vim /etc/privoxy/config
listen-address 127.0.0.1:8118           #8118 是默认端口
forward-socks5t / 127.0.0.1:1080        #转发到本地端口
#我们打开注释就可以,要确保只有一个!

```

## 6.配置环境变量
```
$ vim /etc/profile
export http_proxy=http://127.0.0.1:8118  #这里的端口和上面 privoxy 中的保持一致
export https_proxy=http://127.0.0.1:8118

$ source /etc/profile
```



