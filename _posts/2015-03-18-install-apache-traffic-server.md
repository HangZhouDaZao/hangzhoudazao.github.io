---
layout: post
title: "安装Apache Traffic Server"
description: ""
category: "运维"
tags: [CDN,缓存]
---
{% include JB/setup %}

由于工作原因，我需要考虑缓存方案。后续我想更新缓存和存储架构，缓存内容会大大的加大，所以varnish是一定不适合了（内存缓存严重依赖内存大小）。而squid磁盘缓存根据我已有的经验，性能上确实不是很理想（也可能我配置的问题）。总之，这里想先尝试性的研究下ats，第一步自然是安装。

* **环境:** CentOS release 6.5 (Final),trafficserver-5.2.0
* **参考文档:** [官方文档](https://docs.trafficserver.apache.org/en/latest/getting-started.en.html#installation)
 


具体步骤如下：

<pre class="prettyprint">
#安装依赖
yum install autoconf   automake   pkgconfig   libtool   perl-ExtUtils-MakeMaker   gcc-c++   glibc-devel   openssl-devel   tcl-devel   expat-devel   pcre   pcre-devel   libcap-devel   flex   hwloc-devel   lua-devel
./configure --prefix=/usr/local/ts
make
make check
make install
</pre>

**注意:** 不过根据我在traffic server的一个qq群里获得到的信息来看，社区版本有很多坑，不过我还在学习过程中，先不深入了解了，生产环境可以考虑使用[zym](http://zymlinux.net/)的版本



