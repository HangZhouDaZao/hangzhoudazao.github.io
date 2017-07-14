---
layout: post
title: "nignx优化"
description: "nginx optimization"
category: "运维"
tags: [nginx,optimization]
---
{% include JB/setup %}

nginx优化

# 基本优化

## 1：网络连接的优化：
只能在events模块设置，用于防止在同一一个时刻只有一个请求的情况下，出现多个睡眠进程会被唤醒但只能有一个进程可获得请求的尴尬，如果不优化，在多进程的nginx会影响以部分性能。
```
events {
accept_mutex on; #优化同一时刻只有一个请求而避免多个睡眠进程被唤醒的设置，on为防止被同时唤醒，默认为off，因此nginx刚安装完以后要进行适当的优化。
}
```

## 2.设置是否允许同时接受多个网络连接：
```
events {
accept_mutex on;
multi_accept on; #打开同时接受多个新网络连接请求的功能。
}
```

## 3.隐藏ngxin版本号：
当前使用的nginx可能会有未知的漏洞，如果被黑客使用将会造成无法估量的损失，但是我们可以将nginx的版本隐藏
```
http{
    server_tokens off; 
}
```

## 4.：选择事件驱动模型：
epoll:
.处理新连接事件
.处理定时事件
.处理普通读写事件
.处理从磁盘读事件
```
events {
use epoll; #使用epoll事件驱动，因为epoll的性能相比其他事件驱动要好很多
}
```

## 5：配置单个工作进程的最大连接数：

值不能大于操作系统能打开的最大的文件句柄数，使用ulimit -n可以查看当前操作系统支持的最大文件句柄数，默认为为1024.
```
events {
    worker_connections  102400; #设置单个工作进程最大连接数102400
}
```

## 6：定义MIME-Type：
在浏览器当中可以显示的内容有HTML/GIF/XML/Flash等内容，浏览器为取得这些资源需要使用MIME Type，即MIME是网络资源的媒体类型，Nginx作为Web服务器必须要能够识别全都请求的资源类型，在nginx.conf文件中引用了一个第三方文件，使用include导入：

```
include mime.types;
default_type application/octet-stream;
```

## 7：自定义访问日志：
访问日志是记录客户端即用户的具体请求内容信息，全局配置模块中的error_log是记录nginx服务器运行时的日志保存路径和记录日志的level，因此有着本质的区别，而且Nginx的错误日志一般只有一个，但是访问日志可以在不同server中定义多个，定义一个日志需要使用access_log指定日志的保存路径，使用log_format指定日志的格式，格式中定义要保存的具体日志内容：

```
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;
```

## 8：配置允许sendfile方式传输文件：
　　是由后端程序负责把源文件打包加密生成目标文件，然后程序读取目标文件返回给浏览器；这种做法有个致命的缺陷就是占用大量后端程序资源，如果遇到一些访客下载速度巨慢，就会造成大量资源被长期占用得不到释放（如后端程序占用的CPU/内存/进程等），很快后端程序就会因为没有资源可用而无法正常提供服务。通常表现就是 nginx报502错误，而sendfile打开后配合location可以实现有nginx检测文件使用存在，如果存在就有nginx直接提供静态文件的浏览服务，因此可以提升服务器性能.

　　可以配置在http、server或者location模块，配置如下：
```
sendfile        on;
sendfile_max_chunk 512k;   #Nginxg工作进程每次调用sendfile()传输的数据最大不能超出这个值，默认值为0表示无限制，可以设置在http/server/location模块中。
```

## 9：配置nginx工作进程最大打开文件数：
可以设置为linux系统最大打开的文件数量一致，在全局模块配置
```
worker_rlimit_nofile 65535;
```

## 10：会话保持时间：
```
keepalive_timeout  65 60;  #后面的60为发送给客户端应答报文头部中显示的超时时间设置为60s：如不设置客户端将不显示超时时间。
Keep-Alive:timeout=60  #浏览器收到的服务器返回的报文

如果设置为0表示关闭会话保持功能，将如下显示：
Connection:close  #浏览器收到的服务器返回的报文
```

## 11配置网络监听：
使用命令listen，可以配置监听IP+端口，端口或监听unix socket:
```
listen       8090;   #监听本机的IPV4和IPV6的8090端口，等于listen *:8000
listen       192.168.0.1:8090; #监听指定地址的8090端口
listen     Unix:/www/file  #监听unix socket
```


# sysctl.conf针对IPv4内核的7个参数的配置优化：

## 1、net.core.netdev_max_backlog
每个网络接口的处理速率比内核处理包的速度快的时候，允许发送队列的最大数目。
```
[root@Server1 nginx]# sysctl -a | grep max_backlog
net.core.netdev_max_backlog = 1000
这里默认是1000，可以设置的大一些，比如：
net.core.netdev_max_backlog = 102400
```

## 2、net.core.somaxconn
用于调节系统同时发起的TCP连接数，默认值一般为128，在客户端存在高并发请求的时候，128就变得比较小了，可能会导致链接超时或者重传问题。

```
net.core.somaxconn = 128  
#默认为128，高并发的情况时候要设置大一些，比如：
net.core.somaxconn = 102400
```

## 3、net.ipv4.tcp_max_orphans
设置系统中做多允许多少TCP套接字不被关联到任何一个用户文件句柄上，如果超出这个值，没有与用户文件句柄关联的TCP套接字将立即被复位，同时给出警告信息，这个值是简单防止DDOS（Denial of service）的攻击，在内存比较充足的时候可以设置大一些：

```
net.ipv4.tcp_max_orphans = 32768 
#默认为32768，可以改大一些：
net.ipv4.tcp_max_orphans = 102400
```

## 4、net.ipv4.tcp_max_syn_backlog
用于记录尚未收到客户度确认消息的连接请求的最大值，一般要设置大一些：
```
net.ipv4.tcp_max_syn_backlog = 256  
#默认为256，设置大一些如下：
net.ipv4.tcp_max_syn_backlog =  102400
```

## 5、net.ipv4.tcp_timestamps
用于设置时间戳，可以避免序列号的卷绕，有时候会出现数据包用之前的序列号的情况，此值默认为1表示不允许序列号的数据包，对于Nginx服务器来说，要改为0禁用对于TCP时间戳的支持，这样TCP协议会让内核接受这种数据包，从而避免网络异常，如下：
```
net.ipv4.tcp_timestamps = 1 
#默认为1，改为0，如下：
net.ipv4.tcp_timestamps = 0
```

## 6、net.ipv4.tcp_synack_retries
用于设置内核放弃TCP连接之前向客户端发生SYN+ACK包的数量，网络连接建立需要三次握手，客户端首先向服务器发生一个连接请求，服务器收到后由内核回复一个SYN+ACK的报文，这个值不能设置过多，会影响服务器的性能，还会引起syn攻击：
```
net.ipv4.tcp_synack_retries = 5
#默认为5，可以改为1避免syn攻击
net.ipv4.tcp_synack_retries = 1
```

## 7、net.ipv4.tcp_syn_retries
```
net.ipv4.tcp_syn_retries = 5 
#默认为5，可以改为1
net.ipv4.tcp_syn_retries = 1
```

