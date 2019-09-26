---
layout:     post
title:      Docker安装FastDFS
date:       2019-09-26
author:     JC
header-img: img/golang.jpg
catalog: true
tags:
    - docker
    - fastdfs
---

## 查询镜像

```
docker search fastdfs


NAME                           DESCRIPTION                                     STARS               

OFFICIAL            AUTOMATED

season/fastdfs                 FastDFS                                         47

luhuiguo/fastdfs               FastDFS is an open source high performance d…   19                                      [OK]

morunchang/fastdfs             A FastDFS image                                 12



……
```

采用使用最多的season/fastdfs镜像。

## 下载镜像

	docker image pull season/fastdfs

## 目录配置

在宿主机中新建目录用,于存放fastdfs配置文件和数据。名称根据自己需求，如下只是样例，fastdfs在一台服务器支持多个store_path，每个store_path指向一个存储路径。

```
mkdir /usr/local/fastdfs/etc/ 

mkdir /usr/local/fastdfs/data/storage_data

mkdir /usr/local/fastdfs/data/store_path

mkdir /usr/local/fastdfs/data/tracker_data
```

- etc：配置文件地址
- storage_data：存储数据地址
- tracker_data：存储数据地址
- store_path：扩容

## 获取配置文件

启动一个fastdfs的docker容器，查看容器id，从容器中下载配置文件并且下载到上面创建的 /usr/local/fastdfs/etc/ 目录中.

```
docker run -ti --name fdfs_sh --net=host season/fastdfs sh

docker ps -a   

docker cp -a 07e7af1fdf74:/fdfs_conf/.  /usr/local/fastdfs/etc
```

## 修改配置文件

主要修改的是文件存储目录和跟踪服务器地址，tracker_server 根据自己机器的地址进行配置。

tracker.conf主要修改如下：

```
base_path=/fastdfs/storage

store_path0=/fastdfs/storage/

tracker_server=192.168.1.214:22122
```

storage.conf主要修改如下：

	tracker_server=192.168.1.214:22122

client.conf修改如下：

	tracker_server=192.168.1.214:22122

以上文件，如果需要修改默认端口则也在对应配置文件中进行修改。

	http.tracker_server_port=8880


## 启动容器tracker

启动容器脚本,先启动一个tracker然后再启动一个storage，-v 后面跟的目录映射，TRACKER_SERVER地址根据自己机器的地址进行配置,–privileged=true主要是解决目录权限。

### 启动tracker

	docker run -tid --name  tracker -v /usr/local/fastdfs/data/tracker_data/data:/fastdfs/tracker/data -v /usr/local/fastdfs/etc:/fdfs_conf --privileged=true --net=host  season/fastdfs tracker

### 启动storage

	docker run -tid --name storage -v /usr/local/fastdfs/data/storage_data/data:/fastdfs/storage/data -v /usr/local/fastdfs/data/store_path:/fastdfs/store_path -v /usr/local/fastdfs/etc:/fdfs_conf --privileged=true --net=host -e TRACKER_SERVER:172.17.90.65:22122 season/fastdfs storage

可以使用docker logs容器id查看日志。

## 启动测试

启动一个容器，在容器中进行测试，启动容器会用到client.conf。

	docker run -ti --name fdfs_sh -v /usr/local/fastdfs/etc:/fdfs_conf --privileged=true --net=host season/fastdfs sh

执行完脚本会进入容器内，切换到/usr/bin目录下。

	cd /usr/bin

	fdfs_test  /fdfs_conf/client.conf upload /fdfs_conf/storage.conf

上传完成后会获得文件相关信息：

	example file url: http://192.168.1.214/group1/M00/00/00/wKgB1lozOU-ASzg7AAAgC81RIQ441_big.conf

但此时并没办法在宿主机上进行查看，还需要配置Nginx。

## 安装Nginx

### 下载相关组件

如果未安装git可通过命令安装：

	yum install git

### 下载相关软件。

	git clone https://github.com/happyfish100/libfastcommon.git

	git clone https://github.com/happyfish100/fastdfs-nginx-module.git

	git clone https://github.com/happyfish100/fastdfs.git

### 安装libfastcommon

```
cd libfastcommon/

./make.sh

./make.sh install
```

### 安装fastdfs

Nginx后续要使用到此环境的配置，因此也需安装。
```
./make.sh

./make.sh install
```

安装Nginx依赖
```
yum -y install gcc zlib zlib-devel pcre-devel openssl openssl-devel

wget http://nginx.org/download/nginx-1.15.9.tar.gz

tar -zxvf nginx-1.15.9.tar.gz

cd nginx-1.15.9

## 此处如果需要安装其他插件，比如ssl插件，可类似添加。
./configure --add-module=../fastdfs-nginx-module/src/ --with-http_ssl_module

make

make install
```

安装成功，则/usr/local/目录下就可以看到nginx。

### 异常情况

此过程如果出现异常：

	/usr/local/include/fastdfs/fdfs_define.h:15:27: 致命错误：common_define.h：没有那个文件或目录

则编辑fastdfs-nginx-module/src/config文件，将以下参数的key对应的值修改为：

	ngx_module_incs="/usr/include/fastdfs /usr/include/fastcommon/"

	CORE_INCS="$CORE_INCS /usr/include/fastdfs /usr/include/fastcommon/"

然后再重新执行configure，make等操作。

### 相关配置

拷贝/usr/local/fastdfs/etc/目录下的内容到/etc/fdfs目录下。将fastdfs-nginx-module/src 目录下的mod_fastdfs.conf也复制到/etc/fdfs。

并修改以上配置文件中涉及到的以下参数与实际目录一致。

```
base_path=/usr/local/fastdfs/data/tore_path
store_path0=/usr/local/fastdfs/data/tore_path
tracker_server=192.168.6.78:22122
http.server_port=8880 //需要与nginx监听的端口一致
```

修改 /etc/fdfs/mod_fastdfs.conf：

	url_have_group_name = true //请求路径是否携带组信息 

### 配置Nginx

	vim /usr/local/nginx/conf/nginx.conf

```
server {

    listen       8888;

    server_name  localhost;

    #charset koi8-r;

    #access_log  logs/host.access.log  main;

    location ~/group([0-9])/M00 {

    # root /var/fdfs/storage_path;

        ngx_fastdfs_module;

  }
 ```

当然，对外的端口也可以设置为其他，比如80。

## 使用配置
```
connect_timeout = 60
#网络超时时间
network_timeout = 60
#字符集
charset = UTF-8
#跟踪服务器的端口（默认80端口，可以在storage中配置）
http.tracker_http_port = 8880
http.anti_steal_token = no
http.secret_key = 123456
#跟踪服务器地址 。跟踪服务器主要是起到负载均衡的作用
tracker_server = 47.100.206.217:22122
```
跟踪服务器的端口，默认80端口，可以在storage中配置中的fdsf.conf中配置。

### docker内部命令修改
如果需要修改docker内部的配置文件，需先安装vim命令。

	apt-get update

	apt-get install -y vim

如果执行过程中无法连接，则修改国内镜像源：

	mv /etc/apt/sources.list /etc/apt/sources.list.bak

	echo "deb http://mirrors.163.com/debian/ jessie main non-free contrib" >> /etc/apt/sources.list

	echo "deb-src http://mirrors.163.com/debian/ jessie main non-free contrib" >>/etc/apt/sources.list

然后再执行更新和安装命令即可。

### 开放防火墙

在配置文件中配置涉及到的端口，如果需外网访问则需开放对应的防火墙。



















