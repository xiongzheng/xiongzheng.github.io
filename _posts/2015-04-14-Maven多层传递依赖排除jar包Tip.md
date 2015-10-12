---
layout: post
title: "Maven多层传递依赖排除jar包Tip"
description: ""
category: get
tags: []
---
{% include JB/setup %}


maven传递依赖是罪恶的，常常为了排除某个jar要exclusion上十个jar。这还不算，如果是多层的传递依赖，比如你依赖A，A依赖B，B依赖C，C是个很低版本的包，而且坐标还没有较高的可用版本，导致不能强制指定排除；亦或坐标不同但是包路径完全相同，你需要排除一个。这些情况怎么处理呢？

### 通过maven-enforcer-plugin插件，配置全局排除包
// TODO 自行GG~

### 构建999-not-exist空包【内部常用，推荐】

0.如果有幸mvn repo里已经有了999-not-exist空包，则可以直接跳到第四步骤。
http://mvnrepo.xxx/nexus/index.html#nexus-search;gav~~~999-not-exist~~

这里已经有很多了~


1.构建一个空包pom.xml

```
<?xml version="1.0" encoding="utf-8"?>

<project xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>hessian</groupId>
	<artifactId>hessian</artifactId>
	<version>999-not-exist</version>
</project>
```

2.本地打包
mvn install

3.发布(deploy)刚才构建的jar，发布时，填写坐标，上传空包jar即可。

4.最后在pom中直接指定此版本

```
			<dependency>
				<groupId>hessian</groupId>
				<artifactId>hessian</artifactId>
				<version>999-not-exist</version>
			</dependency>
```

现在打包时会依赖这个空包，但由于里面啥也没有，不会对应用有任何影响；当然如果依赖这个jar的功能，需要引入高版本的jar。

```
			<dependency>
				<groupId>com.caucho</groupId>
				<artifactId>hessian</artifactId>
				<version>4.0.38</version>
			</dependency>
```

