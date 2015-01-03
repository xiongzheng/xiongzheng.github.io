---
layout: post
title: "Groovy引发的PermGen区爆满问题定位与解决"
description: ""
category: question
tags: [groovy]
---
{% include JB/setup %}

### 现象与定位

#### jstack
由于集群有流量的增长，以及开放新的服务出去，所以首先看了一下线程使用情况，未发现问题。
```
$JAVA_HOME/bin/jstack -l PID > jstack.out
grep -A 2 -B 5 -i "com.xxxxxx.xxxx" ./jstack.out
```

#### jstat
使用jstat查看GC情况发现PermGen将满，并且频繁触发FGC，虽然能够回收无效的类，但产生类的速度比FGC的效率还快，直接导致CPU使用率飙升。
```
ps -ef | grep java
$JAVA_HOME/bin/jstat -gcutil PID 1000 100
```

![image](/postimg/permgen-fgc1.png)
图1 PermGen区升到95%引发频繁FGC

#### jmap & MAT
遇到线上FGC问题常用的工具是用jmap & eclipse MAT来分析JVM内存使用情况了。网上资料很多，这里不在赘述，需要注意的一点是jmap是会触发"stop the world"的，所以最好是拉出来，然后做dump操作。如果用eclipse MAT分析切记提前把eclipse最大能使用的内存调整下，否则分析上G的dump文件会挂掉。

![image](/postimg/groovy-classload1.png)
图2 Dump中的类加载器情况

由于是PermGen区的泄漏，通过分析发现类加载部分有大量的GroovyClassLoader，这时想到有使用两个服务，内部实现是使用Groovy配置的方式(这里我用Groovy封装了一个服务提供的微框架)。但这两个服务其实使用的并不频繁，所以导致切换后的现象不是雪崩式的集群挂掉，而是慢慢的，逐步有机器load飙高告警。

### 解决方法
定位到问题，事情就好办了，首先看一下处理Groovy加载执行的代码；然后为生成的类加一层对象缓存；由于脚本中用到了Binding上下文对象，为了线程安全性，调整执行时的方式。最后问题得到解决。

#### 原始的调用方式
```
Object scriptObject = null;
try {
	Binding binding = new Binding();
	binding.setVariable("context", this.context);
	binding.setVariable("clientInfo", clientInfo);
	binding.setVariable("params", params);
	binding.setVariable("data", data);
	 	
	GroovyShell shell = new GroovyShell(binding);
	scriptObject = (Object) shell.evaluate(script);
} catch (Throwable t) {
	log.error("groovy script eval error. script: " + script, t);
}

return scriptObject;
```

#### 为Groovy Script增加缓存
```
private Map<String, Object> scriptCache = new ConcurrentHashMap<String, Object>();
...

Object scriptObject = null;
try {
	Binding binding = new Binding();
	binding.setVariable("context", this.context);
	binding.setVariable("clientInfo", clientInfo);
	binding.setVariable("params", params);
	binding.setVariable("data", data);
	
	Script shell = null;
	if (isCached(cacheKey)) {
		shell = (Script) getCaches().get(cacheKey);
	} else {
		shell = new GroovyShell().parse(script);
	}
	
	shell.setBinding(binding);
	scriptObject = (Object) shell.run();
	
	// clear
	binding.getVariables().clear();
	binding = null;
	
	// Cache
	if (!isCached(cacheKey)) {
		shell.setBinding(null);
		getCaches().put(cacheKey, shell);
	}
} catch (Throwable t) {
	log.error("groovy script eval error. script: " + script, t);
}

return scriptObject;
```

#### 解决Binding的线程安全问题
```
private Map<String, Object> scriptCache = new ConcurrentHashMap<String, Object>();
...

Object scriptObject = null;
try {
	Binding binding = new Binding();
	binding.setVariable("context", this.context);
	binding.setVariable("clientInfo", clientInfo);
	binding.setVariable("params", params);
	binding.setVariable("data", data);
	
	Script shell = null;
	if (isCached(cacheKey)) {
		shell = (Script) getCaches().get(cacheKey);
	} else {
		shell = cache(cacheKey, script);
	}
	
	scriptObject = (Object) InvokerHelper.createScript(shell.getClass(), binding).run();
	
	// Cache
	if (!isCached(cacheKey)) {
		getCaches().put(cacheKey, shell);
	}
} catch (Throwable t) {
	log.error("groovy script eval error. script: " + script, t);
}

return scriptObject;
```

### 小结
这次碰到的问题还是很具有典型性的，其中的思路和用到的工具可以阅读这两本JVM相关的书籍获取：《深入理解Java虚拟机》和《Java性能优化权威指南》。

