---
layout: post
title: "Nutch添加中文分词器"
description: ""
category: question
tags: []
---
{% include JB/setup %}



#### 0.准备工作,JDK6+,安装ant1.7.1+,安装javacc

```
tar xzvf apache-ant-1.7.1-bin.tar.gz -C /usr/local/
tar xzvf javacc-5.0.tar.gz -C /usr/local/
```
 
#### 1.安装Nutch

```
tar xzvf nutch-1.0.tar.gz -C /usr/local/
```
 
#### 2.测试中文分词
```
export LANG=zh_CN.GB18030 #此设置可使命令行输入的中文被NutchAnalysis正常读取

cd /usr/local/nutch-1.0/

java -classpath nutch-1.0.jar:./conf/:./lib/lucene-core-2.4.0.jar:./lib/hadoop-0.19.1-core.jar:./lib/commons-logging-1.0.4.jar:./lib/log4j-1.2.15.jar org.apache.nutch.analysis.NutchAnalysis
```

简写：

```
java -classpath nutch-1.0.jar:./conf/:./lib/* org.apache.nutch.analysis.NutchAnalysis
```

```
Query: 人逢喜事精神爽
人 逢 喜 事 精 神 爽
```

这里输入“人逢喜事精神爽”，从输出的结果可以看出Nutch默认处理是一元分词。

#### 3.加入Paoding中文分词

```
cd /usr/local/nutch-1.0/src/java/org/apache/nutch/analysis/
 
vim NutchDocumentAnalyzer.java
```

修改：

```
  /** Returns a new token stream for text from the named field. */
  public TokenStream tokenStream(String fieldName, Reader reader) {
    Analyzer analyzer;
    /*
    if ("anchor".equals(fieldName))
      analyzer = ANCHOR_ANALYZER;
    else
      analyzer = CONTENT_ANALYZER;
    */
 
    analyzer = new net.paoding.analysis.analyzer.PaodingAnalyzer();
 
    return analyzer.tokenStream(fieldName, reader);
  }
```

拷贝paoding-analysis.jar到/usr/local/nutch-1.0/lib
 
打包nutch
/usr/local/apache-ant-1.7.1/bin/ant jar
会在/usr/local/nutch-1.0/build目录下生成新的nutch-1.0.jar
 
测试新的nutch-1.0.jar

```
cd /usr/local/nutch-1.0/

## 导入庖丁词典路径
export PAODING_DIC_HOME=/dic

## 注意这里使用./build/下的新包
java -classpath ./build/nutch-1.0.jar:./conf/:./lib/* org.apache.nutch.analysis.NutchAnalysis

```

```
Query: 人逢喜事精神爽
人 逢 喜 事 精 神 爽
```

这里nutch还是用了一元分词，继续下一步。
 
#### 4.修正nutch先使用一元分词，然后才将结果传递给分词器问题。

```
cd /usr/local/nutch-1.0/src/java/org/apache/nutch/analysis

## 修改NutchAnalysis.jj文件
vim NutchAnalysis.jj
```
```
将 “src/java/org/apache/nutch/analysis/NutchAnalysis.jj” line 130:
| <SIGRAM: <CJK> >
改为:
| <SIGRAM: (<CJK>)+ >
```

运行javacc生成java源文件

```
cd /usr/local/nutch-1.0/src/java/org/apache/nutch/analysis
/usr/local/javacc-5.0/bin/javacc NutchAnalysis.jj
```

重新打包nutch

```
cd /usr/local/nutch-1.0/
/usr/local/apache-ant-1.7.1/bin/ant jar
```

再测试新的nutch-1.0.jar

```
export PAODING_DIC_HOME=/dic
java -classpath ./build/nutch-1.0.jar:./conf/:./lib/* org.apache.nutch.analysis.NutchAnalysis
```
```
Query: 人逢喜事精神爽
"喜事 人逢喜事 精神 精神爽"
```

从结果可以看出已使用庖丁分词器分词了。

