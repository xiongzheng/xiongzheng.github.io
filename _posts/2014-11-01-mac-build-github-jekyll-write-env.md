---
layout: post
title: "Mac下搭建github+jekyll写作环境"
description: ""
category: get 
tags: [github,jekyll]
---
{% include JB/setup %}

## 升级Command Line
It seems like you have to install the Xcode Command Line Tools first (run xcode-select --install in your terminal).

## 更新RubyGems到最新版本
https://rubygems.org/pages/download

```
$ gem update --system          # may need to be administrator or root
$ gem install rubygems-update  # again, might need to be admin/root
$ update_rubygems              # ... here too
```

## 安装jekyll
```
$ gem install jekyll           # may need to be administrator or root
```

## clone jekyll-bootstrap
```
$ git clone https://github.com/plusjade/jekyll-bootstrap.git
$ cd jekyll-bootstrap
$ jekyll serve
```

## 替换jquery链接
下载一个jquery压缩版放入/assets/javascript/jquery-1.11.1.min.js

然后将_includes/themes/bootstrap-3/default.html中的

```
https://ajax.googleapis.com/ajax/libs/jquery/1.10.2/jquery.min.js
```

替换为：

```
/assets/javascript/jquery-1.11.1.min.js
```

## 调整首页
//TODO

## 开始写作
```
rake post title="Hello World"
```

## 中文支持

将jekyll默认的markdown解析引擎maruku 更换为RDiscount

sudo gem install rdiscount
然后在_config.yml中 pygments: true之前加一行markdown: rdiscount,这样再次生成网站的时候就使用RDiscount来解析md文件了。

## 个性化域名
USERNAME.github.com 其实是个挺不错的域名了，但是如果想绑定自己的域名，可以在USERNAME.github.com文件夹下新建一个CNAME 文件，里面的内容就是你的域名，比如USERNAME.COM

然后按照GitHub所说将你的域名指向204.232.175.78这个ip，例如在GoDaddy中，添加一条@记录。这样当再次访问你的域名的时候，就会只想GitHub Pages了（可能DNS更新会需要一段时间）。

## favicon
如果想要个性化的标签页图标，也就是所说的favicon，可以将选定的favicon.ico 放在USERNAME.github.com文件夹根目录下就可以了。

## 安装&更换theme
```
rake theme:install git="https://github.com/jekyllbootstrap/theme-the-program.git"

rake theme:switch name="the-program"
```
