---
layout: post
title: "Nginx+Tomcat+Terracotta的Web服务器集群实做"
description: ""
category: get
tags: []
---
{% include JB/setup %}



#### 1.准备工作
两个Linux服务器,可以用VMware装一个,然后配置好再克隆一个,修改IP即可。
Host1：192.168.0.79
Host2：192.168.0.80
先配置好jdk1.6.0和tomcat6。Host1上还将配置Nginx(负载均衡),Terracotta(session集群).

#### 2.安装Terracotta
下载Terracotta的包，

```
http://d2zwv9pap9ylyd.cloudfront.net/terracotta-3.4.1-installer.jar    带安装功能的包
http://d2zwv9pap9ylyd.cloudfront.net/terracotta-3.4.1.tar.gz    不带安装的包
http://d2zwv9pap9ylyd.cloudfront.net/terracotta-3.4.1-src.tar.gz    源码包
```

这里使用terracotta-3.4.1.tar.gz

```
tar zxvf terracotta-3.4.1.tar.gz
mv terracotta-3.4.1 /usr/local/terracotta
```

#### 3.配置Tomcat作为Terracotta的客户端

```
复制/usr/local/terracotta/sessions/terracotta-session-1.1.1.jar
/usr/local/terracotta/common/terracotta-toolkit-1.1-runtime-2.1.0.jar这两个jar到Tomcat对应目录。
Tomcat 5.0 and 5.5    对应目录 $CATALINA_HOME/server/lib
Tomcat 6.0        对应目录 $CATALINA_HOME/lib
```

编辑/usr/apache-tomcat-6.0.14/conf/context.xml文件

```
<Context>
<!- - 增加此配置 -->
<Valve className="org.terracotta.session.TerracottaTomcat60xSessionValve" tcConfigUrl="localhost:9510"/>
</Context>
```

如上是Tomcat6的配置方式，其他WebServer配置方式可见参考文档的具体说明。9510端口为Terracotta的通讯端口，另外默认9520为它的管理端口稍后用到。

#### 4.启动单Terracotta，测试。
启动Terracotta

```
[Host1]# /usr/local/terracotta/bin/start-tc-server.sh &
```

启动Tomcat

```
[Host1]# service tomcatd start
[Host2]# service tomcatd start
```

(
注意：之前Host2处的Tomcat context.xml配置需要修改为

```
<Valve className="org.terracotta.session.TerracottaTomcat60xSessionValve" tcConfigUrl="192.168.0.79:9510"/>
```

即指向启动的Terracotta
)

启动Windows下的Terracotta开发控制台(Terracotta Developer Console)，此控制台使用9520端口来监控Terracotta,这个用terracotta-3.4.1-installer.jar带安装的软件包可以在Windows下安装或tar解压运行也可。

```
[PROMPT] ${TERRACOTTA_HOME}\bin\dev-console.bat
```

(
注意：需要配置Linux主机名才能远程监控的到。
设置主机名(如果要永久修改RedHat的hostname，就修改/etc/sysconfig/network文件，将里面的HOSTNAME这一行修改成HOSTNAME=NEWNAME)

```
编辑/etc/hosts
::1     localhost.localdomain   localhost
192.168.0.79 Host1  #添加此行
然后service network restart，
查验hostname:
[root@localhost ~]# hostname
host1
```

通过Terracotta Developer Console查看连接的Terracotta Server和Tomcat Client是否正常。

#### 5.安装、配置Nginx
下载http://nginx.org/en/download.html
安装Nginx: (预先需要安装：rpm -ivh pcre-devel-6.6-2.el5_1.7.i386.rpm)

```
[Host1]#tar zxvf nginx-0.8.54.tar.gz
[Host1]#cd nginx-0.8.54
[Host1]#./configure --with-http_stub_status_module
[Host1]#make;make install
```

简单配置：修改/usr/local/nginx/conf/nginx.conf

```
...
http {
    ...
    upstream mysvr {
        server 192.168.0.79:8080;
        server 192.168.0.80:8080;
    }

    server {
        listen       80;
        server_name  192.168.0.79;

        location / {
            proxy_pass http://mysvr;
            ....
        }
   	...
    }
}
...
```

```
测试配置文件
/usr/local/nginx/sbin/nginx -t

启动Nginx
/usr/local/nginx/sbin/nginx
```

#### 6.观察集群session
jsp验证页test.jsp,部署到两个Tomcat的ROOT项目下:

```
<%@ page session="true" %>
<html>
<head>
    <title>test Host1</title> <!-- //Host2就写为"test Host1" 以示区分 -->
</head>
<body>
<%
    out.println("Session Id:"+request.getSession().getId()+"<br />");
    out.println("Creation Time:"+request.getSession().getCreationTime());

    String name=(String)session.getAttribute("name");
    if(name==null||name.equals("")){
        session.setAttribute("name","Hello Host1!"); //Host2就写为"Hello Host2!"
        out.println(session.getAttribute("name"));
    }else{
        out.println(name);
    }
%>
</body>
</html>
```

访问test.jsp

```
http://192.168.0.79/test.jsp
```

刷新多次。发现除session的id号被Terracotta加了个类似版本编号的后缀有所区别,
例如：Session Id:2jUFdCvDTqrZViJXzvh8.3和Session Id:2jUFdCvDTqrZViJXzvh8.2,以外其他的输出都是一致的。
已达到session集群的作用。

如果关闭Terracotta，然后注释/usr/apache-tomcat-6.0.14/conf/context.xml文件的Valve,重启两个tomcat，再次访问，并刷新，会发现sessionid及其内容在跳变，没有集群特性。这就是Terracotta的作用。

另外有两个单点故障的点，Nginx和Terracotta，Nginx可以通过keepalived配置为热备;
Terracotta内置了热备方式，可以在Host2上也启用Terracotta，需要tc-config.xml配置，具体可参见官方文档。

#### 参考文档：
http://www.terracotta.org/documentation/product-documentation-1page.html 5 Clustering Web Applications with Terracotta Web Sessions 章节

---

#### 配置Terracotta双机热备

```
[Host1]#/bin/start-tc-server.sh -f /usr/local/terracotta/tc-config.xml -n Server1 &
[Host2]#/bin/start-tc-server.sh -f /usr/local/terracotta/tc-config.xml -n Server2 &
```

tc-config.xml 样例：

```
<?xml version="1.0" encoding="UTF-8"?>

<tc:tc-config 
xsi:schemaLocation="http://www.terracotta.org/schema/terracotta-5.xsd"
xmlns:tc="http://www.terracotta.org/config"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

  <servers>

    <server host="192.168.0.79" name="Server1">

      <data>%(user.home)/terracotta/server-data</data>

      <logs>%(user.home)/terracotta/server-logs</logs>

    </server>

    <server host="192.168.0.80" name="Server2">

      <data>%(user.home)/terracotta/server-data</data>

      <logs>%(user.home)/terracotta/server-logs</logs>

    </server>

    <ha>

       <mode>networked-active-passive</mode>

       <networked-active-passive>

               <election-time>5</election-time>

       </networked-active-passive>

    </ha>

  </servers>

  <clients>

    <logs>%(user.home)/terracotta/client-logs</logs>

  </clients>

</tc:tc-config>
```
