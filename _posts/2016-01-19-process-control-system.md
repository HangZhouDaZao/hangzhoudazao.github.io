---
layout: post
title: "process control system"
description: ""
category: "运维"
tags: [运维]
---
{% include JB/setup %}

## daemontools

**参考文献:** [daemontools](http://cr.yp.to/daemontools.html)

### 安装


	wget http://cr.yp.to/daemontools/daemontools-0.76.tar.gz
	tar xf daemontools-0.76.tar.gz
	cd admin/daemontools-0.76
	./package/install


安装完毕后会生成/command，/service两个目录。

**注意：** 安装过程中如果出现以下错误，


	/usr/bin/ld: errno: TLS definition in /lib64/libc.so.6 section .tbss mismatches non-TLS reference in envdir.o


把src/error.h中的extern int errno删除，添加


	#include <errno.h>


即可解决

### 配置启动

#### 一个简单的例子

假设有两个服务，启动方式分别为/root/program1  2345和/root/program2
2346

* 启动daemontools服务，执行 /command/svscanboot &
* 创建service目录,创建启动脚本

如下：


	#mkdir -p myservice/{program1,program2}
	#cat > myservice/program1/run
	#!/bin/sh
	exec /root/program1  2345
	#cat > myservice/program2/run
	#!/bin/sh
	exec /root/program2  2346
	#chmod 755 myservice/program1/run
	#chmod 755 myservice/program2/run


*   创建软连接，daemontools会自动启动服务

如下:


	$ps aux | grep [p]rogram
	$ ln -s /root/myservice/program1/ /service/
	$ ln -s /root/myservice/program2/ /service/
	$ ps aux | grep [p]rogram
	root     16942  0.0  0.0   4172   360 pts/0    S    14:27   0:00 supervise program1
	root     16943  0.0  0.0   4296   356 pts/0    S    14:27   0:00 /root/program1 2345
	root     16947  0.0  0.0   4172   356 pts/0    S    14:27   0:00 supervise program2
	root     16948  0.0  0.0   4296   360 pts/0    S    14:27   0:00 /root/program2 2346
	$ pstree -a -p 16700
	svscanboot,16700 /command/svscanboot
	  ├─readproctitle,16703 service errors:...
	  └─svscan,16702 /service
	      ├─supervise,16942 program1
	      │   └─program1,16943 2345
	      └─supervise,16947 program2
	          └─program2,16948 2346
	$ kill 16943
	$ pstree -a -p 16700
	svscanboot,16700 /command/svscanboot
	  ├─readproctitle,16703 service errors:...
	  └─svscan,16702 /service
	      ├─supervise,16942 program1
	      │   └─program1,16964 2345
	      └─supervise,16947 program2
	          └─program2,16948 2346


#### 其他配置

* 配置nobody执行程序

当然首先应用程序的权限要开放给nobody


	#!/bin/sh
	exec setuidgid nobody  /tmp/program1  2345


* 日志

其实不是日志，而是把应用程序的stdout和stderr内容打到日志

1. 在run脚本里添加exec 2>&1，使得应用程序的stderr也输出到stdout
2. 立目录log，在log目录里建立main目录
3.  在log目录里添加run脚本

内容如下：


	#!/bin/bash
	umask 0027
	exec multilog ./main


这样日志会打到main目录里面的current文件,不过感觉这种日志其实没什么卵用。

### 运维命令

- **svc -u:** 启动服务
- **svc -d:** 停止服务
- **svstat:** 查看服务状态
- **svc -dx:** 删除服务

如下：

	# pstree -a -p 16700
	svscanboot,16700 /command/svscanboot
	  ├─readproctitle,16703 service errors:...
	  └─svscan,16702 /service
	      ├─supervise,16942 program1
	      │   └─program1,17051 2345
	      └─supervise,16947 program2
	          └─program2,16948 2346
	# svstat /service/program1
	/service/program1: up (pid 17051) 78 seconds
	# svc -d /service/program1/
	# pstree -a -p 16700
	svscanboot,16700 /command/svscanboot
	  ├─readproctitle,16703 service errors:...
	  └─svscan,16702 /service
	      ├─supervise,16942 program1
	      └─supervise,16947 program2
	          └─program2,16948 2346
	# svstat /service/program1
	/service/program1: down 12 seconds, normally up
	# svc -u /service/program1/
	# pstree -a -p 16700
	svscanboot,16700 /command/svscanboot
	  ├─readproctitle,16703 service errors:...
	  └─svscan,16702 /service
	      ├─supervise,16942 program1
	      │   └─program1,17076 2345
	      └─supervise,16947 program2
	          └─program2,16948 2346
	# rm  /service/program1
	# svc -dx /root/myservice/program1
	# pstree -a -p 16700
	svscanboot,16700 /command/svscanboot
	  ├─readproctitle,16703 service errors:...
	  └─svscan,16702 /service
	      └─supervise,16947 program2
	          └─program2,16948 2346


### 有哪些坑

各个run脚本里最好显示的配置好关键的环境变量，比如max open files,因为子
进程继承父进程的环境变量，而daemontools默认的环境变量可能并不符合你的
需求。

实际工作中有过一次血的教训，一个高并发长连接的服务端程序，同时维
持多个socket连接，通过daemontools监管，而daemontools开机启动时默认的
Max open files是1024，结果导致了服务端程序出现了异常。

## Supervisor

**参考文献:** [supervisord](http://supervisord.org/)

supervisor的文档比较丰富，建议使用时阅读官网
 手册。

### 安装


	easy_install supervisor


### 配置启动

supervisor可以用不同的配置起多个服务，这里简单的起一份。

首先生成默认配置文件


	echo_supervisord_conf > supervisord.conf


修改一下


	[supervisord]
	logfile=/tmp/supervisord.log ; 日志文件
	logfile_maxbytes=50MB        ; 超过50M就滚动日志
	logfile_backups=10           ; 滚动日志会删除，这里保留最新10份
	loglevel=info                ; 日志级别
	pidfile=/tmp/supervisord.pid ; supervisord的pid
	nodaemon=false               ; 是否前台运行
	minfds=1024                  ; 这个好啊，意思是当max open file小于1024时，会设置为1024，当设置不上去了，就直接报错
	minprocs=200                 ;类似同上
	[program:mytestcat]          ;mytestcat实际上是命令的组名了
	command=/bin/cat
	process_name=%(program_name)s_%(process_num)02d
	numprocs=2
	[program:mytestserver]
	command=/root/program1 2345


启动 supervisord


	supervisord -c supervisord.conf


## 其他还需了解的

* systemd
* SysV 和 LSB
* Linux init过程
