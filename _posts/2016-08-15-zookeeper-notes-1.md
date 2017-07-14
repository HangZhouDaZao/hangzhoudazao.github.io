---
layout: post
title: "zookeeper notes (1)"
description: "zookeeper notes"
category: "zookeeper"
tags: [hadoop,zookeeper,distributed system]
---
{% include JB/setup %}

ZooKeeper是一个分布式的，开放源码的分布式应用程序协调服务，许多分布式开源系统依赖于ZooKeeper。
本文不讨论ZooKeeper的细节，只是基于Fabric来演示怎么部署安装一个简单的ZooKeeper集群。

详细内容请移步 [zookeeper官网](https://zookeeper.apache.org/)

## 前置条件

机器列表：

<pre class="prettyprint">
10.13.11.21 test1121
10.13.11.22 test1122
10.13.11.23 test1123
10.13.11.24 test1124
10.13.11.25 test1125
</pre>


## 配置


<pre class="prettyprint">
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/cluster/zookeeper/data
clientPort=2181
server.1=test1121:2888:3888
server.2=test1122:2888:3888
server.3=test1123:2888:3888
server.4=test1124:2888:3888
server.5=test1125:2888:3888
</pre>

## 安装

fabfile见如下,发布机器中/home/admin/zookeeper为zookeeper所在目录，此fabfile将把zookeeper安装到/cluster目录下。myid根据脚本中的zookeeper_myid_dict分配。
<pre class="prettyprint">
from fabric.api import *

host_deploy = ['10.13.11.21','10.13.11.22','10.13.11.23','10.13.11.24','10.13.11.25']
host_deploy3 = ['10.13.11.21','10.13.11.22','10.13.11.23']
host_deploy2 = ['10.13.11.24','10.13.11.25']

env.port = 22
env.password = 'xxx' #-I
env.user = 'xxx'  #-u
env.roledefs = { #-R
    'deploy': host_deploy,
    'deploy3': host_deploy3,
    'deploy2': host_deploy2
}

zookeeper_myid_dict = {
	'10.13.11.21': '1',
	'10.13.11.22': '2',
	'10.13.11.23': '3',
	'10.13.11.24': '4',
	'10.13.11.25': '5'
}

source_root='/home/admin/'
dest_root='/cluster/'

@parallel
def remove_zookeeper():
    sudo("su admin -c '"+dest_root+"/zookeeper/bin/zkServer.sh stop'")
    sudo("rm -rf "+dest_root+"/zookeeper*")


@parallel
def upload_zookeeper():
    put(source_root+'/software/zookeeper-3.4.8.tar.gz','/tmp/zookeeper-3.4.8.tar.gz',mode=0644)
    sudo('mkdir -p '+dest_root)
    sudo("rm -rf "+dest_root+"/zookeeper")
    sudo('tar xf /tmp/zookeeper-3.4.8.tar.gz -C '+dest_root)
    sudo('chown admin '+dest_root+' -R')
    sudo("su admin -c 'ln -s "+dest_root+"/zookeeper-3.4.8 "+dest_root+"/zookeeper'")


def prepare_local(ip):
    local("rm -rf "+ip)
    local("mkdir "+ip)
    with lcd(ip):
      local("cp -rf "+source_root+"/zookeeper/conf/ conf")
      local("tar czf zookeeper_conf.tgz conf")


@parallel
def sync_zookeeper():
    myid=zookeeper_myid_dict[env.host]
    prepare_local(env.host)
    put(env.host+'/zookeeper_conf.tgz','/tmp/zookeeper_conf.tgz',mode=0644)
    sudo("su admin -c 'tar xf /tmp/zookeeper_conf.tgz -C "+dest_root+"/zookeeper/'")
    sudo("su admin -c 'mkdir -p "+dest_root+"/zookeeper/data'")
    sudo("su admin -c 'echo "+myid+" >"+dest_root+"/zookeeper/data/myid'")


@parallel
def start_zookeeper():
    with cd(dest_root+"/zookeeper/"):
      sudo("su admin -c 'sh "+dest_root+"/zookeeper/bin/zkServer.sh start'")

@parallel
def stop_zookeeper():
    with cd(dest_root+"/zookeeper/"):
      sudo("su admin -c 'sh "+dest_root+"/zookeeper/bin/zkServer.sh stop'")
</pre>

如何安装
<pre class="prettyprint">
fab -R deploy upload_zookeeper
fab -R deploy sync_zookeeper
fab -R deploy start_zookeeper
</pre>

## 其他

* 扩容，缩容，更改机器看起来比较麻烦，至少客户端的配置，弄得不好就要全部更新一把
* 为了处理当机（更换机器）这些问题，给客户端最好使用域名
* 扩容缩容，如果服务可以停止一段时间，那么基本上就是更改配置文件和myid，重启即可
* 迁移集群，可以采取先扩容，再缩容的方式
