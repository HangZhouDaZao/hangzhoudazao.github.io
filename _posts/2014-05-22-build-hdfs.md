---
layout: post
title: " HDFS(采用HA和Federation)搭建"
description: ""
category: "hadoop"
tags: [存储,hadoop]
---
{% include JB/setup %}


本次操作目标是搭建一个HDFS集群，且此集群采用HA和Federation。

### 硬件环境

真是硬件环境为一台服务器，为了模拟分布式集群，采用kvm虚拟出9台虚拟机。分别为：

 <pre class="prettyprint">
10.0.5.31 zk1
10.0.5.32 zk2
10.0.5.33 zk3
10.0.5.41 namenode1
10.0.5.42 namenode2
10.0.5.43 namenode3
10.0.5.44 namenode4
10.0.5.51 datanode1
10.0.5.52 datanode2
10.0.5.53 datanode3
</pre>

* zk1、zk2和zk3用作zookeeper和JournalNode集群
* namenode1和namenode2组成一组HA，namenode3和namenode4组成一组HA
* datanode1、datanode2和datanode3为存储节点

### 基本配置

* 关闭所有服务器的iptables
* 所以服务器配置好hosts，如上所示
* 配置好免密码登陆

java环境变量

<pre class="prettyprint">
export JAVA_HOME=/root/java
export PATH=$JAVA_HOME/bin:$PATH
export CLASSPATH=$CLASSPATH:$JAVA_HOME/lib:$JAVA_HOME/jre/lib
</pre>

#### 目录配置

10台机器基本上是一样的虚拟机，这里基本共享配置。

* namenode存储目录`/root/hadoop/data/nn_data/`
* datanode存储目录`/root/hadoop/data/dn_data1/`,这里只配置一个
* journal存储目录`/root/hadoop/data/journaldata/`
* zookeeper存储目录 `/root/hadoop/data/zkdata/`

#### 软件版本

* 操作系统 CentOS release 6.4 (Final)
* java 1.7.0_45
* zookeeper-3.4.6
* hadoop-2.0.0-cdh4.4.0



### zookeeper集群

#### 配置

zoo.cfg

<pre class="prettyprint">
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/root/hadoop/data/zkdata
clientPort=2181
server.1=zk1:2888:3888
server.2=zk2:2888:3888
server.3=zk3:2888:3888
</pre>
在/root/hadoop/data/zkdata下配置myid


#### 启动

<pre class="prettyprint">
./zkServer.sh  start
</pre>


### hadoop配置

#### core-site.xml


<?prettify?>

	<?xml version="1.0" encoding="UTF-8"?>
	<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
	<configuration>
	<property>
	<name>fs.defaultFS</name>
	<value>viewfs://nsX</value>
	</property>
	<property>
	<name>fs.viewfs.mounttable.nsX.link./c1</name>
	<value>hdfs://cluster1/tmp</value>
	</property>
	<property>
	<name>fs.viewfs.mounttable.nsX.link./c2</name>
	<value>hdfs://cluster2/tmp2</value>
	</property>
	</configuration>	
	
	
	


#### hdfs-site.xml


<?prettify?>

	<?xml version="1.0" encoding="UTF-8"?>
	<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
	<configuration>
	   <property>
	      <name>dfs.namenode.name.dir</name>
	      <value>/root/hadoop/data/nn_data/</value>
	   </property>
	   <property>
	      <name>dfs.nameservices</name>
	      <value>cluster1,cluster2</value>
	   </property>
	   <property>
	      <name>dfs.ha.namenodes.cluster1</name>
	      <value>nn1,nn2</value>
	   </property>
	   <property>
	      <name>dfs.namenode.rpc-address.cluster1.nn1</name>
	      <value>namenode1:8020</value>
	   </property>
	   <property>
	      <name>dfs.namenode.rpc-address.cluster1.nn2</name>
	      <value>namenode2:8020</value>
	   </property>
	   <property>
	      <name>dfs.namenode.http-address.cluster1.nn1</name>
	      <value>namenode1:50070</value>
	   </property>
	   <property>
	      <name>dfs.namenode.http-address.cluster1.nn2</name>
	      <value>namenode2:50070</value>
	   </property>
	   <property>
	      <name>dfs.ha.namenodes.cluster2</name>
	      <value>nn3,nn4</value>
	   </property>
	   <property>
	      <name>dfs.namenode.rpc-address.cluster2.nn3</name>
	      <value>namenode3:8020</value>
	   </property>
	   <property>
	      <name>dfs.namenode.rpc-address.cluster2.nn4</name>
	      <value>namenode4:8020</value>
	   </property>
	   <property>
	      <name>dfs.namenode.http-address.cluster2.nn3</name>
	      <value>namenode3:50070</value>
	   </property>
	   <property>
	      <name>dfs.namenode.http-address.cluster2.nn4</name>
	      <value>namenode4:50070</value>
	   </property>
	   <property>
	      <name>dfs.datanode.data.dir</name>
	      <value>/root/hadoop/data/dn_data1/</value>
	   </property>
	   <property>
	      <!-- 每个nameserver 不一样！！！！-->
	      <name>dfs.namenode.shared.edits.dir</name>
	      <value>qjournal://zk1:8485;zk2:8485;zk3:8485/cluster1</value>
	   </property>
	   <property>
	      <name>dfs.journalnode.edits.dir</name>
	      <value>/root/hadoop/data/journaldata/</value>
	   </property>
	   ￼
	   <property>
	      <name>dfs.client.failover.proxy.provider.cluster1</name>
	      <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
	   </property>
	   <property>
	      <name>dfs.client.failover.proxy.provider.cluster2</name>
	      <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
	   </property>
	   <property>
	      <name>dfs.ha.fencing.methods</name>
	      <value>sshfence(root:22)</value>
	   </property>
	   <property>
	      <name>dfs.ha.fencing.ssh.private-key-files</name>
	      <value>/root/.ssh/id_rsa</value>
	   </property>
	   <property>
	      <name>dfs.ha.fencing.ssh.connect-timeout</name>
	      <value>30000</value>
	   </property>
	   <property>
	      <name>dfs.ha.automatic-failover.enabled</name>
	      <value>true</value>
	   </property>
	   <property>
	      <name>ha.zookeeper.quorum</name>
	      <value>zk1:2181,zk2:2181,zk3:2181</value>
	   </property>
	   <property>
	      <name>fs.trash.interval</name>
	      <value>360</value>
	   </property>
	   <property>
	      <name>dfs.replication</name>
	      <value>3</value>
	   </property>
	</configuration>
	





### JournalNode集群

<pre class="prettyprint">
sbin/hadoop-daemon.sh start journalnode
</pre>

### namenode集群


首先格式化集群，在主namenode上

<pre class="prettyprint">
./bin/hdfs namenode -format 
./bin/hdfs namenode -initializeSharedEdits -force
./sbin/hadoop-daemon.sh start namenode
</pre>

在次namenode上

<pre class="prettyprint">
bin/hdfs namenode -bootstrapStandby
sbin/hadoop-daemon.sh start namenode
</pre>


##### 配置自动切换

首先格式化zk

<pre class="prettyprint">
bin/hdfs zkfc -formatZK
</pre>

然后在所有namenode上启动

<pre class="prettyprint">
./hadoop/hadoop-2.0.0-cdh4.4.0/sbin/hadoop-daemon.sh start zkfc
</pre>


启动第二组HA时，唯一的区别是namenode format时指定集群id，例如

<pre class="prettyprint">
bin/hdfs namenode -format -clusterid CID-5c5d754c-20f6-43b6-bf16-e2239e93dbb7
</pre>

### datanode配置

<pre class="prettyprint">
./hadoop/hadoop-2.0.0-cdh4.4.0/sbin/hadoop-daemon.sh start datanode
</pre>





## 一些烂坑

* 配置文件不能用ip，只能用hostname
* hostname不能含有下划线









  















