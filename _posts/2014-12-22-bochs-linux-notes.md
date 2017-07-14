---
layout: post
title: "bochs上调试linux内核的笔记"
description: ""
category: "linux"
tags: [bochs,linux]
---
{% include JB/setup %}

bochs是一个x86虚拟机，可以完全模拟x86机器。本文是我在学习《linux内核完全注释[^1]》的前期准备。本文内容不断更新。

[^1]: <http://oldlinux.org/book_cn.html>

## 环境准备

* Linuxmint 17
* 安装 bochs-sdl，bochs-x ，bochs

## 第一次启动

下载《linux内核完全注释》的第17章中的[例子](http://oldlinux.org/Linux.old/bochs/linux-0.11-devel-050518.zip),里面有作者制作好的内核镜像文件和根文件系统盘。每个文件的作用在书中都有详细介绍，这里不在赘述。

不过，作者提供的配置文件是基于bochs2.2的，而linuxmint上直接安装的bochs版本是2.4.6，bochs不同版本配置不兼容，所以作者的配置在这里不可以用。这里提供一份可用的配置文件，内容如下：

<pre class="prettyprint">
#first.bxrc
romimage: file=$BXSHARE/BIOS-bochs-latest 
megs: 16
vgaromimage: file=$BXSHARE/VGABIOS-lgpl-latest
floppya: 1_44="bootimage-0.11-hd", status=inserted
ata0-master: type=disk, path="hdc-0.11-new.img", mode=flat, cylinders=410, heads=16, spt=38
boot: a
log: bochsout.txt
display_library: sdl
</pre>

使用命令`bochs -f first.bxrc`即可启动虚拟机。

## 和虚拟机交换代码

接上面的例子继续。和bochs虚拟机中的linux系统交换代码的关键就是挂载它的minix文件系统，这里要挂载的就是hdc-0.11-new.img。书中第17.3.2章给出了具体的方法，这里只给出我的操作过程示范：


<pre class="prettyprint">
fengya-VirtualBox linux-0.11-devel-050518 # losetup /dev/loop0 hdc-0.11-new.img 
fengya-VirtualBox linux-0.11-devel-050518 # fdisk /dev/loop0

Command (m for help): x

Expert command (m for help): p

Disk /dev/loop0: 16 heads, 38 sectors, 410 cylinders

Nr AF  Hd Sec  Cyl  Hd Sec  Cyl     Start      Size ID
 1 00   0   3    0  15  38  203          2     124030 81
 2 00   0   1  204  15  38  407     124032     124032 81
 3 00   0   0    0   0   0    0          0          0 00
 4 00   0   0    0   0   0    0          0          0 00


Expert command (m for help): q
fengya-VirtualBox linux-0.11-devel-050518 # losetup -d /dev/loop0
fengya-VirtualBox linux-0.11-devel-050518 # losetup -o 1024 /dev/loop0 hdc-0.11-new.img 
fengya-VirtualBox linux-0.11-devel-050518 # mount -t minix /dev/loop0 test/

fengya-VirtualBox linux-0.11-devel-050518 # ls test/
bin  dev  etc  image  Image  linux-0.00  mnt  shoelace  tmp  usr  var                                                                                                               
</pre>


test目录挂载了hdc-0.11-new.img中的分区，这样就可以交换数据了。

## 编译、替换内核

继续接上面的例子。先备份bootimage-0.11-hd，因为后面旧内核会被编译的内核给覆盖。

这里用的书中第17.4中的[例子](http://oldlinux.org/Linux.old/bochs/linux-0.00-050613.zip)。把例子中的内核代码linux-0.00拷贝到hdc-0.11-new.img，这里我选择拷贝到根目录/linux-0.00。然后启动bochs，执行

<pre class="prettyprint">
make
make disk                                                                                                          
</pre>

就完成了变异替换了内核。重启后将会看到屏幕上不停的打印字符，这是linux-0.00的功能。




## 参考资料

* [linux内核完全注释官网](http://oldlinux.org)
* [bochs官网](http://bochs.sourceforge.net)

## Footnote


