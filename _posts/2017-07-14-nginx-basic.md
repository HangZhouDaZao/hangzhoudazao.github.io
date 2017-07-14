---
layout: post
title: "nignx基本配置"
description: "nginx basic"
category: "运维"
tags: [nginx]
---
{% include JB/setup %}

nignx配置

nignx配置

## nginx 配置文件基本结构
    全局块
    event{} event块
    http{   http块
        server{ server块
            location{} location块
        }
        
    }
    
全局块：配置影响nginx全局的指令，一般有：
```
error_log  logs/error.log :日志存放路径
user nobody :运行nginx服务器的用户组
pid logs/nginx.pid :nginx进程pid存放路径
```
    
event块：配置影响nginx服务器与用户的网络连接，一般有：
```
worker_connections  1024 :每个进程的最大连接数
```
http块：可以嵌套多个server，配置代理，缓存，日志等绝大多数功能和第三方模块设置

server块：配置虚拟主机的相关参数，一个http 可以存放多个server

location块：配置路由请求，以及各种页面的处理情况
    
下面是例子
```
#user sjd sjds;  #配置用户或者组，默认为nobody nobody
#worker_processes 2;  #允许生成的进程数，默认为1
#pid /nginx/pid/nginx.pid;   #指定nginx进程运行文件存放地址
error_log log/error.log debug;  #制定日志路径，级别。这个设置可以放入全局块，http块，server块，级别以此为：debug|info|notice|warn|error|crit|alert|emerg
events{
    accept_mutex on;   #设置网路连接序列化，防止惊群现象发生，默认为on
    multi_accept on;  #设置一个进程是否同时接受多个网络连接，默认为off
    #use epoll;      #事件驱动模型，select|poll|kqueue|epoll|resig|/dev/poll|eventport
    worker_connections  1024;    #最大连接数，默认为512
}

http{
     include       mime.types;   #文件扩展名与文件类型映射表
     default_type  application/octet-stream; #默认文件类型，默认为text/plain
     #access_log off; #取消服务日志 
     log_format myFormat '$remote_addr–$remote_user [$time_local] $request $status $body_bytes_sent $http_referer $http_user_agent $http_x_forwarded_for'; #自定义格式
     access_log log/access.log myFormat;  #combined为日志格式的默认值
     sendfile on;   #允许sendfile方式传输文件，默认为off，可以在http块，server块，location块。
     sendfile_max_chunk 100k;  #每个进程每次调用传输数量不能大于设定的值，默认为0，即   不设上限。
     keepalive_timeout 65;  #连接超时时间，默认为75s，可以在http，server，location块。
     error_page 404 https://www.baidu.com; #错误页
     server{
         keepalive_requests 120; #单连接请求上限次数。
         listen       4545;   #监听端口
         server_name  127.0.0.1;   #监听地址
         location  ~*^.+$ {       #请求的url过滤，正则匹配，~为区分大小写，~*为不区分大小写。
         
           #root path;  #根目录
           #index vv.txt;  #设置默认页
           proxy_pass  http://mysvr;  #请求转向mysvr 定义的服务器列表
           deny 127.0.0.1;  #拒绝的ip
           allow 172.18.5.54; #允许的ip           
         }
     }
     
}
```
	