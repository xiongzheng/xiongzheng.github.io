---
layout: post
title: "HBase单机及伪分布环境搭建"
description: ""
category: question
tags: []
---
{% include JB/setup %}


#### 一.单机模式
##### 1.安装HBase

```
tar -xzvf hbase-0.92.1.tar.gz
```
　

##### 2.配置参数
修改hbase-site.xml：

```
<configuration>
       <property>  
              <name>hbase.rootdir</name>  
              <value>/home/xiongzheng/hadoop/hbase-0.92.1/data</value>  
        </property>  
</configuration>
```
　

##### 3.配置/etc/hosts , 将127.0.0.1改为本地ip

```
10.17.12.95	localhost
10.17.12.95	xiongzheng-Lenovo
```
　

##### 4.启动HBase

```
bin/start-hbase.sh
```
　

##### 5.简单操作

```
bin/hbase shell
```


(1).建立表格user_info，以及两个列族 k、v

```
create 'user_info','k','v' 
```


(2).查看表

```
list
```


(3) 查看表结构

```
describe 'user_info'
```


(4) 插入行 put 表名，行名，列族名：列名标签，值

```
put 'user_info','memberId123','v:IP','127.0.0.1' 
```


(5) 查询表数据 get 表名，行名

```
get 'user_info','memberId123'
```


(6) 全表查询

```
scan 'user_info'
```


(7) 查看表中某列族所有数据

```
scan 'user_info',{COLUMNS => 'v'}
```


(8) 删除表

```
disable 'user_info'
drop 'user_info'
```
　

#### 二.伪分布式运行模式

##### 1.安装HBase

```
tar -xzvf hbase-0.92.1.tar.gz
```
　

##### 2.配置参数
编辑hbase-0.92.1/conf/hbase-env.sh，添加环境变量

```
export JAVA_HOME=/System/Library/Frameworks/JavaVM.framework/Versions/1.6.0/Home 
export HBASE_CLASSPATH=~/Hadoop-1.0.3/conf  
```


注意：HBASE_CLASSPATH的值是Hadoop_HOME目录下的conf目录

编辑hbase-0.92.1/conf/hbase-site.xml，按如下配置

```
	<configuration>    
	    <property>    
	        <name>hbase.rootdir</name>    
	        <value>hdfs://localhost:9000/hbase</value>    
	    </property>    
	    <property>    
	        <name>hbase.cluster.distributed</name>    
	        <value>true</value>    
	    </property>    
	</configuration>  
```

注意：hbase.rootdir的value中，hdfs://localhost:9000是Hadoop配置文件core-site.xml中fs.default.name的值。

　

##### 3.先启动Hadoop，启动HBase

```
hbase-0.92.1/bin/start-hbase.sh  
```

