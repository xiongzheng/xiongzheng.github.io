## 在 windows(cygwin) 或 mac 下安装git、node、ruby等前端开发环境

yy一段全栈。
// TODO

安装列表：
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

// TODO 配置用户名、邮箱及ssh-key

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

> Mac系统自带~


### Sass、Compass
Sass、Compass是CSS的一组扩展，让CSS变得如动态语言一般强大，有了上面的ruby安装器，安装起来也相当方便。
```
gem install sass
gem install compass
```

### 开始写代码
