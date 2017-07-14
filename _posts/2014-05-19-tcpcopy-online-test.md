---
layout: post
title: "利用TCPCopy做真实流量测试"
description: ""
category: "软件测试"
tags: [网络,压力测试]
---
{% include JB/setup %}

TCPCopy的作用正如其名字所示，用于把线上TCP流量copy到测试服务器上，用真实的流量来做测试，以减少或避免失误。

这里是一个简单示例（采用tcpcopy的新方式），具体请参见[tcpcopy](https://github.com/wangbin579/tcpcopy)官方文档。未来如果需要更高级的应用，将会补充该博客。

## 实验环境

* 客户端，实际生产环境中可能是nginx代理

<pre class="prettyprint">
10.0.5.13 qionghua
10.0.5.18 shushan
</pre>


* 线上服务器,这里的服务用python -m SimpleHTTPServer模拟

<pre class="prettyprint">
10.0.5.14 tianhe
10.0.5.15 lingsha
</pre>

* 测试服务器,企图用新机器ziying和mengli替换掉tianhe和lingsha，同样的，启动python -m SimpleHTTPServer

<pre class="prettyprint">
10.0.5.16 ziying
10.0.5.17 mengli
</pre>

* 协助服务器,tcpcopy新架构中需要一台服务器协助测试

<pre class="prettyprint">
10.0.5.19 wangshu
</pre>

* 为了防止干扰，我关掉了所有服务器的的iptables

## 过程

目标使用ziying替换tianhe，mengli替换lingsha。

在lingsha和tianhe处，安装tcpcopy客户端

<pre class="prettyprint">
./configure --enable-advanced  
make && make install
</pre>

tianhe启动tcpcopy

<pre class="prettyprint">
tcpcopy -x 10.0.5.14:8000-10.0.5.16:8000 -s 10.0.5.19  -d
</pre>

lingsha启动tcpcopy

<pre class="prettyprint">
tcpcopy -x 10.0.5.15:8000-10.0.5.17:8000 -s 10.0.5.19  -d
</pre>

ziying、mengli设置路由

<pre class="prettyprint">
route add -host 10.0.5.13 gw 10.0.5.19
route add -host 10.0.5.18 gw 10.0.5.19
</pre>


wanghsu协助服务器安装intercept（tcpcopy server）

<pre class="prettyprint">
yum install libpcap-devel
./configure --enable-advanced -enable-pcap
make && make install
</pre>

启动

<pre class="prettyprint">
intercept -i eth0 -F 'tcp and src host (10.0.5.16 or 10.0.5.17) and src port 8000' -d
</pre>

这时候，在qionghua和shushan上对tianhe和lingsha做curl（端口默认8000），在ziying和mengli处将会收到同样的连接。

## 结论
以上叙述都是“知其然而不知其所以然”，虽然我知道每一步大概都做了什么，但是如果涉及到TCP/IP层，稍微细节一点的，我却还是不能完全说清楚。基础还是不够。
