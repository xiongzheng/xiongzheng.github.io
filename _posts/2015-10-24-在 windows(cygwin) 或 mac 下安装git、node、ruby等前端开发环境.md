---
layout: post
title: "在 windows(cygwin) 或 mac 下安装git、node、ruby等前端开发环境"
description: ""
category: get
tags: []
---
{% include JB/setup %}


我记得每一年技术部年会都会由老大说出一个技术主题，之前有服务化、模块化、工具化、数据化等等等等，今年的主题是全栈。遥想当年，在供职过的两家公司什么抗机器、装机架；什么安装系统、加固、搭建Web或任务应用的前后端环境、打包发布、网络配置；什么JavaScript、CSS、VB、.NET、PHP、JAVA等等语言；什么Mootools、JQuery、Ext、Lucene、Solr、Hibernate、Spring；什么Oracle、Mysql、SQLServer、MongoDB、PL/SQL -- 我都耍的有模有样。然后加入阿里，在感叹分工明确、精益求精之余在专业上深入挖掘，再然后要全栈了，不免有些五味杂陈，当然之前也谈不上全栈，我理解的全栈不是Java写的比前端好，CSS写的比后端好，而是各个方面都比较深入，且能快速学习新的理论与技术。提到搞全栈，不由得想到两个正负能量的词："剩余价值"、"终身学习"，于我，后者想的多一点。

跑偏了，好了，接下来我们就来看一下如何搭建前端开发环境并将你的代码推送到线上。

前端开发环境软件安装列表：
- 命令行环境
- Git
- Node.js (NPM)
- Grunt
- TNPM、DEF
- Ruby (RubyGems)
- Sass、Compass


### 命令行环境
Windows下建议使用Cygwin来构建命令行环境，省力省资源。

安装Cygwin要注意必须用32位的版本：https://www.cygwin.com/setup-x86.exe。如果是64位版本在使用RubyGems的时候会卡住，如下：
```
gem sources --add https://ruby.taobao.org/
      0 [main] ruby 9156 D:\cygwin64\bin\ruby.exe: *** fatal error - NtCreateEvent(lock): 0xC0000058
Hangup
```

我安装Cygwin的姿势：
1.分两个文件夹，一个cygwinsetup用于存放安装器(setup-x86.exe)和下载的临时安装文件、一个cygwin是实际命令行的路径。
2.点开setup-x86.exe，一路Next：选择在线安装 -> 填写根路径cygwin -> 填写下载的临时安装文件保存路径cygwinsetup -> 选择镜像站点(选国内163或阿里云)。
3.来到了选择安装包页面上，如果你硬盘足够大可以不管三七二一，全部安装，点击列表ALL旁边的Default将其切换到Install状态。当然，这要花上很漫长的时间，所以还是稍后按需获取吧，跳过选择安装包的话会安装基本的环境，继续一路Next完成安装。

> Mac 自带。


### Git
在Cygwin里安装Git你可以通过Linux命令行方式源码三步走安装(configure;make;make install)，或者通过Cygwin的安装器安装也是很方便的。
再次打开Cygwin的cygwinsetup/setup-x86.exe，到选择安装包页面，然后选择如下安装包(上面选择了ALL Install的土财请绕过)：
- git
- git-completion
- vim
- openssh

配置用户名、邮箱及ssh-key，gitlib可以设置commit者必须是注册用户，所以这里的用户名和邮箱地址必须同gitlib的注册用户名和邮箱。
```
git config --global user.name "${yourname}"
git config --global user.email "${yourname}@alibaba-inc.com"

\## 生成ssh-key
ssh-keygen -t rsa -C "${yourname}@alibaba-inc.com"
\## 将~/.ssh/id_rsa.pub中的文件内容copy至 http://gitlab.alibaba-inc.com/profile/keys ，你就可以开始使用gitlib了。
```


> Mac下如果已有MacPorts则 
> ```
> port selfupdate
> port install git-core
> ```
> 如果有Homebrew则 ```brew install git```；
> 如果啥没有可以去下个dmg傻瓜包： ```http://sourceforge.net/projects/git-osx-installer/```；
> 或者装个Github客户端： ```https://desktop.github.com/```，妥妥的。


### Node.js (NPM)
Node.js在Windows下的极速安装方式只需3步： 1.下载```https://nodejs.org/dist/v4.2.1/node-v4.2.1-x64.msi```。2.一路Next。3.终。

使用Cygwin安装选择如下安装包：
- Devel->gcc-g++
- Devel->gcc-mingw-g++
- Devel->gcc4-g++
- Devel->git
- Devel->make
- Devel->openssl-devel
- Devel->pkg-config
- Devel->zlib-devel
- Editor->vim
- Python->全部
- Web->curl
- Web->wget

然后下载 ``` https://nodejs.org/dist/v4.2.1/node-v4.2.1-linux-x86.tar.gz ``` ，解压 ```tar xzvf node-v4.2.1-linux-x86.tar.gz```，命令行三步走安装。

> Mac下两种方式，都仅需一步： ```port install nodejs``` OR ```brew install node```


### Grunt
有了Node.js后，自带了NPM包管理器，安装Grunt就很方便了，Windows下和Mac下一样，风一般的感觉：
```
npm install -g grunt-cli
```

### TNPM、DEF
安装淘宝前端集成开发环境，同理两个命令搞定：
```
npm install -g tnpm --registry=http://registry.npm.alibaba-inc.com  # 安装 tnpm
tnpm install @ali/def -g  # 安装 def 客户端
```

### Ruby (RubyGems)
Cygwin默认安装自带。

由于众所周知的原因，默认的Ruby安装包管理器RubyGems的默认地址https://rubygems.org/是不能用的。
需要更新一下安装资源地址到淘宝镜像：
```
gem sources --add https://ruby.taobao.org/ --remove https://rubygems.org/
gem sources -l
```

> Mac系统也自带~


### Sass、Compass
Sass、Compass是CSS的一组扩展，让CSS变得如动态语言一般强大，有了上面的ruby安装器，安装起来也相当方便。
```
gem install sass
gem install compass
```


### 开始写代码
以一个简单的PC工程为例，H5需要kimi环境参见这里：http://www.atatech.org/articles/40818

1. clone
```
git clone git@gitlab.alibaba-inc.com:${project-path}.git
```

2. 创建分支
```
git checkout -b daily/${version}
```

3. grunt打包
```
grunt // 如果上面的插件安装正确，则这里grunt会顺利返回Success；如果还缺少一些需要的工具可以通过tnpm安装。
```

4. status、diff、add、commit
```
git status // 查看当前状态，哪些变更或新增
git diff // 查看变更
git add src/${name}.js // add文件，注意将build目录下的js压缩文件变更也一并add，不要遗漏
git commit -m "${info}" // 提交变更
```

5. 部署daily
```
git push origin daily/${version}
```

6. 正式发布
```
git tag publish/${version}
git push origin publish/${version}
```

发完收工，发布仅需半分钟，这酸爽~


### 总结
费的神马劲儿，且用Mac！



























