---
layout: post
title: "simple hadoop cluster"
description: "simple hadoop cluster"
category: "hadoop"
tags: [hadoop,MapReduc,Yarn]
---
{% include JB/setup %}

本文演示搭建一个简单hadoop集群的过程，差不多参考了官网的 [Hadoop Cluster Setup](http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-common/ClusterSetup.html)。

这里使用的版本是 Apache Hadoop 2.7.2

## 前置条件

机器列表：

<?prettify?>

	10.13.11.21 test1121
	10.13.11.22 test1122
	10.13.11.23 test1123
	10.13.11.24 test1124
	10.13.11.25 test1125


jdk安装路径：
<?prettify?>

	/usr/java/jdk1.8.0_60/


运维机器的hadoop解压在：
<?prettify?>

	/home/admin/hadoop


## 配置

###  配置Hadoop Daemons环境变量

etc/hadoop/hadoop-env.sh

<?prettify?>

	export JAVA_HOME=/usr/java/jdk1.8.0_60/


etc/hadoop/yarn-env.sh

<?prettify?>

	export JAVA_HOME=/usr/java/jdk1.8.0_60/


etc/hadoop/mapred-env.sh

<?prettify?>

	export JAVA_HOME=/usr/java/jdk1.8.0_60/


### 配置Hadoop Daemons

etc/hadoop/core-site.xml
<?prettify?>

	<configuration>
	<property>
	 <name>fs.defaultFS</name>
	 <value>hdfs://test1121:8020</value>
	</property>
	</configuration>


etc/hadoop/hdfs-site.xml

<?prettify?>

	<configuration>
	<property>
	<name>dfs.nameservices</name>
	<value>testcluster1</value>
	</property>
	<property>
	<name>dfs.ha.namenodes.testcluster1</name>
	<value>nn1</value>
	</property>
	<property>
	<name>dfs.namenode.rpc-address.testcluster1.nn1</name>
	<value>test1121:8020</value>
	</property>
	<property>
	<name>dfs.namenode.http-address.testcluster1.nn1</name>
	<value>test1121:50070</value>
	</property>
	<property>
	<name>dfs.namenode.name.dir</name>
	<value>/cluster/hadoop/hadoop_nn_data</value>
	</property>
	<property>
	<name>dfs.datanode.data.dir</name>
	<value>/cluster/hadoop/hadoop_dn_data</value>
	</property>
	</configuration>


etc/hadoop/yarn-site.xml
<?prettify?>

	<configuration>
	<property>
	<name>yarn.nodemanager.aux-services</name>
	<value>mapreduce_shuffle</value>
	</property>
	<property>
	<name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
	<value>org.apache.hadoop.mapred.ShuffleHandler</value>
	</property>
	<property>
	<name>yarn.resourcemanager.address</name>
	<value>test1121:8032</value>
	</property>
	<property>
	<name>yarn.resourcemanager.scheduler.address</name>
	<value>test1121:8030</value>
	</property>
	<property>
	<name>yarn.resourcemanager.resource-tracker.address</name>
	<value>test1121:8035</value>
	</property>
	<property>
	<name>yarn.resourcemanager.admin.address</name>
	<value>test1121:8033</value>
	</property>
	<property>
	<name>yarn.resourcemanager.webapp.address</name>
	<value>test1121:8088</value>
	</property>
	</configuration>


etc/hadoop/mapred-site.xml
<?prettify?>

	<configuration>
	<property>
	<name>mapreduce.framework.name</name>
	<value>yarn</value>
	</property>
	</configuration>



## 启动

使用Fabric来运维集群，fabfile如下：

<?prettify?>

	from fabric.api import *
	
	host_deploy = ['10.13.11.21','10.13.11.22','10.13.11.23','10.13.11.24','10.13.11.25']
	host_hadoop_namenode = ['10.13.11.21']
	host_hadoop_datanode = ['10.13.11.22','10.13.11.23','10.13.11.24','10.13.11.25']
	
	env.port = 10022
	env.password = 'xxxx' #-I
	env.user = 'xxxx'  #-u
	env.roledefs = { #-R
	    'deploy': host_deploy,
	    'hadoop_namenode' : host_hadoop_namenode,
	    'hadoop_datanode': host_hadoop_datanode
	}
	
	source_root='/home/admin/'
	dest_root='/cluster/'
	
	@parallel
	def upload_hadoop():
	    put('/home/admin/software/hadoop-2.7.2.tar.gz','/tmp/hadoop-2.7.2.tar.gz',mode=0777)
	    sudo('mkdir -p '+dest_root)
	    sudo("rm -rf "+dest_root+"/hadoop")
	    sudo('tar xf /tmp/hadoop-2.7.2.tar.gz -C '+dest_root)
	    sudo('chown admin '+dest_root+' -R')
	    sudo("su admin -c 'ln -s "+dest_root+"/hadoop-2.7.2 "+dest_root+"/hadoop'")
	
	
	
	@parallel
	def remove_hadoop():
	    sudo("rm -rf "+dest_root+"/hadoop*")
	
	
	def prepare_local(ip):
	    local("rm -rf "+ip)
	    local("mkdir "+ip)
	    with lcd(ip):
	      local("cp -rf "+source_root+"/hadoop/etc/ etc/")
	      local("tar czf hadoop_conf.tgz etc/")
	
	@parallel
	def sync_hadoop():
	    put('/etc/hosts','/tmp/hosts',mode=0644)
	    sudo("mv /tmp/hosts /etc/hosts")
	    prepare_local(env.host)
	    put(env.host+'/hadoop_conf.tgz','/tmp/hadoop_conf.tgz',mode=0644)
	    sudo("su admin -c 'tar xf /tmp/hadoop_conf.tgz -C "+dest_root+"/hadoop'")
	    sudo("su admin -c 'mkdir -p "+dest_root+"/hadoop/hadoop_nn_data'")
	    sudo("su admin -c 'mkdir -p "+dest_root+"/hadoop/hadoop_dn_data'")
	
	def format_hadoop():
	    sudo("su admin -c '"+dest_root+"/hadoop/bin/hdfs namenode -format '")
	
	def start_namenode():
	    sudo("su admin -c '"+dest_root+"/hadoop/sbin/hadoop-daemon.sh start namenode '")
	
	def start_datanode():
	    sudo("su admin -c '"+dest_root+"/hadoop/sbin/hadoop-daemon.sh start datanode '")
	
	def start_resourcemanager():
	    sudo("su admin -c '"+dest_root+"/hadoop/sbin/yarn-daemon.sh start resourcemanager '")
	
	def start_nodemanager():
	    sudo("su admin -c '"+dest_root+"/hadoop/sbin/yarn-daemon.sh start nodemanager '")


按以下命令执行:
<?prettify?>

	fab -R deploy upload_hadoop #上传压缩包，创建目录，解压
	fab -R deploy sync_hadoop # 同步hadoop配置文件，同步hosts文件，创建测试用的namenode和datanode数据目录
	fab -R hadoop_namenode format_hadoop #格式化
	fab -R hadoop_namenode start_namenode
	fab -R hadoop_datanode start_datanode
	fab -R hadoop_namenode start_resourcemanager
	fab -R hadoop_datanode start_nodemanager


### 其他
* RM本身就启动了一个WebAppProxy
* MapReduce Job History Server 先不管

## 测试

跑一遍 [官网教程](http://hadoop.apache.org/docs/current/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html) 里的WordCount通过即可
