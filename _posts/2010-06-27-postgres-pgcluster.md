---
layout: post
title: "postgres pgcluster 配置"
description: ""
category: get
tags: []
---
{% include JB/setup %}


在一台linux服务器上配置postgres数据库同步复制。也可以分多台配置，一台复制服务器(rep1)、两个pg数据库(cluster2、cluster3)。
 
#### 一.设置Linux静态IP和主机名。

> 如果是多台，此处的主机名配置视情况而定，只要rep1、cluster2、cluster3互相能用主机名ping通就可以了。

1.1. 修改配置文件

```
vi /etc/sysconfig/network-scripts/ifcfg-eth0

DEVICE=eth0
ONBOOT=yes
\#BOOTPROTO=dhcp
BOOTPROTO=static
IPADDR=192.168.1.50
NETMASK=255.255.255.0
GATEWAY=192.168.1.1
```

1.2. 使IP地址生效：

```
service network restart
```

1.3.  设置主机名(如果要永久修改RedHat的hostname，就修改/etc/sysconfig/network文件，将里面的HOSTNAME这一行修改成HOSTNAME=NEWNAME)
编辑/etc/hosts

```
::1     localhost.localdomain   localhost
192.168.1.50 rep1.localdomain rep1 cluster2 cluster3  #添加此行
```
 
#### 二.编译、安装pgcluster 

> 源码安装postgres8.1+pgcluster1.5

2.1. 下载postgresql-8.1对应的pgcluster-1.5.0rc21.tar.gz (包含postgresql-8.1版本与pgcluster-1.5.0插件)

2.2. ```tar xzf pgcluster-1.5.0rc21.tar.gz```

2.3. 安装

```
cd pgcluster-1.5.0rc21
./configure --prefix=/var/lib/pgsql;make;make install
```

2.4. 添加postgres用户

```
useradd postgres
passwd postgres
chown -R postgres /var/lib/pgsql
chgrp -R postgres /var/lib/pgsql
```

2.5. 初始化数据库

```
su postgres
mkdir /var/lib/pgsql/data2 /var/lib/pgsql/data3
/var/lib/pgsql/bin/initdb -D /var/lib/pgsql/data2
(/var/lib/pgsql/bin/initdb -D /var/lib/pgsql/data3)
```

2.6. 允许网络访问pg，修改/var/lib/pgsql/data2 和/var/lib/pgsql/data3目录下的配置，注意端口不同。
修改/var/lib/pgsql/data/pg_hba.conf

```
\#IPv4 (此处必须为信任trust)
host　all　　all　192.168.0.0/24　　trust
修改/var/lib/pgsql/data/postgresql.conf
listen_addresses = '*'
port = 5002
(port = 5003)
```


#### 三.配置集群
3.1.

```
su postgres
cd /var/lib/pgsql/data2
cp /var/lib/pgsql/share/cluster.conf.sample cluster.conf
```

在cluster2上面建立/var/lib/pgsql/data2/cluster.conf文件如下：

```
<Replicate_Server_Info>
 <Host_Name>rep1</Host_Name>
 <Port>8001</Port>
 <Recovery_Port>8101</Recovery_Port>
</Replicate_Server_Info>
 
<Host_Name>cluster2</Host_Name>
<Recovery_Port>7002</Recovery_Port>
<Rsync_Path>/usr/bin/rsync</Rsync_Path>
<Rsync_Option>ssh -1</Rsync_Option>
<Rsync_Compress>yes</Rsync_Compress>
<Rsync_Timeout>10min</Rsync_Timeout>
<Rsync_Bwlimit>0KB</Rsync_Bwlimit>
<Pg_Dump_Path>/var/lib/pgsql/bin/pg_dump</Pg_Dump_Path>
<Ping_Path>/bin/ping</Ping_Path>
<When_Stand_Alone>read_write</When_Stand_Alone>
<Replication_Timeout>1min</Replication_Timeout>
<LifeCheck_Timeout>3s</LifeCheck_Timeout>
<LifeCheck_Interval>11s</LifeCheck_Interval>
```

3.2.

```
cd /var/lib/pgsql/data3
cp /var/lib/pgsql/share/cluster.conf.sample cluster.conf
```
 
在cluster3上面建立/var/lib/pgsql/data3/cluster.conf文件如下(注意名称和Recovery_Port与cluster2要不同)：

```
<Replicate_Server_Info>
 <Host_Name>rep1</Host_Name>
 <Port>8001</Port>
 <Recovery_Port>8101</Recovery_Port>
</Replicate_Server_Info>
 
<Host_Name>cluster3</Host_Name>
<Recovery_Port>7003</Recovery_Port>
<Rsync_Path>/usr/bin/rsync</Rsync_Path>
<Rsync_Option>ssh -1</Rsync_Option>
<Rsync_Compress>yes</Rsync_Compress>
<Rsync_Timeout>10min</Rsync_Timeout>
<Rsync_Bwlimit>0KB</Rsync_Bwlimit>
<Pg_Dump_Path>/var/lib/pgsql/bin/pg_dump</Pg_Dump_Path>
<Ping_Path>/bin/ping</Ping_Path>
#此处设置为读写，当此数据库单独存在时也可写入，如果是read_only,只有这一台存在时，只能查不能改
<When_Stand_Alone>read_write</When_Stand_Alone>
<Replication_Timeout>1min</Replication_Timeout>
<LifeCheck_Timeout>3s</LifeCheck_Timeout>
<LifeCheck_Interval>11s</LifeCheck_Interval>
```
 
3.3.

```
cd /var/lib/pgsql/share
cp pgreplicate.conf.sample pgreplicate.conf
```
 
在rep1上面建立/var/lib/pgsql/etc/pgreplicate.conf文件，内容如下：

```
<Cluster_Server_Info>
 <Host_Name>cluster2</Host_Name>
 <Port>5002</Port>
 <Recovery_Port>7002</Recovery_Port>
</Cluster_Server_Info>
<Cluster_Server_Info>
 <Host_Name>cluster3</Host_Name>
 <Port>5003</Port>
 <Recovery_Port>7003</Recovery_Port>
</Cluster_Server_Info>
 
<Host_Name>rep1</Host_Name>
<Replication_Port>8001</Replication_Port>
<Recovery_Port>8101</Recovery_Port>
<RLOG_Port>8301</RLOG_Port>
<Response_Mode>normal</Response_Mode>
<Use_Replication_Log>no</Use_Replication_Log>
<Replication_Timeout>1min</Replication_Timeout>
<LifeCheck_Timeout>3s</LifeCheck_Timeout>
<LifeCheck_Interval>15s</LifeCheck_Interval>
``` 
 
#### 四.启动集群，下面的启动顺序很重要(su postgres)
4.1. 启动replication服务器(rep1)：
```
/var/lib/pgsql/bin/pgreplicate -lnv -D /var/lib/pgsql/share
        (停止 ctrl+c 或 /var/lib/pgsql/bin/pgreplicate -D /var/lib/pgsql/share stop)
```
 
4.2. 分别启动数据库cluster2, cluster3：
```
/var/lib/pgsql/bin/pg_ctl -D /var/lib/pgsql/data2 start
/var/lib/pgsql/bin/pg_ctl -D /var/lib/pgsql/data3 start
(停止 /var/lib/pgsql/bin/pg_ctl -D /var/lib/pgsql/data2 stop)
```
 
停止集群的顺序正好相反。
 
 
#### 五.测试数据同步
在cluster2, cluster3两个数据库上做创建、插入操作。
create table test_cluster (id serial,"name" varchar(255));
insert into test_cluster("name") values('hello');
 
 
#### 附录
1.如果启动出现"could not translate host name localhost"问题,则修改/etc/hosts加入 127.0.0.1 localhost.localdomain localhost , 然后重启network。

2.复制服务器不停的报 PGRcreateConn():Retry. h_errno is 1,reason is 'fe_sendauth: no password supplied 错误，
则检查data2和data3目录下的/var/lib/pgsql/data/pg_hba.conf文件需要将rep1的ip设置为trust，rep1复制服务器会用postgres角色对cluster2和cluster3进行无密码的访问。

```
\#IPv4
host　all　　all　192.168.0.0/24　　trust
```
 
3.如果是多台服务器，需配置SSH无密码互访
3.1. vi /etc/ssh/sshd_config
a、打开AuthorizedKeysFile，就是删除前面的#号 (如果有的话)
b、添加允许访问的帐户，AllowUsers postgres root

3.2. 通过ssh-keygen产生RSA公私密钥对(密码为空)(rep1，cluster2，cluster3都要运行)

```
usermod -d /var/lib/pgsql/ postgres
su postgres
ssh-keygen -t rsa -P ""
```

这样会在/var/lib/pgsql/.ssh/下生成id_rsa和id_rsa.pub

3.3. 将公钥进行相互拷贝(顺序执行)

```
rep1：
[postgres@rep1 ~]$scp -r /var/lib/pgsql/.ssh/id_rsa.pub postgres@cluster2:/var/lib/pgsql/.ssh/id_rsa.pub.rep1
将rep1的公钥拷贝到cluster2中，并将其重命名为id_rsa.pub.rep1。
从rep1远程登录到cluster2(ssh cluster2)，执行命令：
[postgres@cluster2 ~]$cd /var/lib/pgsql/.ssh
[postgres@cluster2 ~/.ssh]$cat id_rsa.pub.rep1 >> authorized_keys
 
cluster2:
[postgres@cluster2 ~]$scp -r /var/lib/pgsql/.ssh/id_rsa.pub postgres@rep1:/var/lib/pgsql/.ssh/id_rsa.pub.cluster2
将cluster2的公钥拷贝到rep1中，并将其重命名为id_rsa.pub.cluster2。
远程登录到rep1(ssh rep1)，执行命令：
[postgres@rep1 ~]$cd /var/lib/pgsql/.ssh
[postgres@rep1 ~/.ssh]$cat id_rsa.pub.cluster2 >> authorized_keys
 
... cluster3 ...
```

3.4. 重启ssh服务
```
su root
service sshd restart
```

3.5. 互访测试，如无需密码则完成配置。
```
[postgres@rep1 ~]$ssh postgres@cluster2
[postgres@cluster2 ~]$ssh postgres@rep1
```
