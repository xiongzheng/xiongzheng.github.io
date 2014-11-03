---
layout: post
title: "Mac下搭建github+jekyll写作环境"
description: ""
category: get 
tags: [github,jekyll]
---
{% include JB/setup %}

> 之前有在公共博客或是专业的技术博客上写写文章、笔记。后来由于从事安全相关行业，转而把一些工作中的总结、笔记记录在了OneNote上，纯离线不对外开放。但这种转换导致文章产出率很低，常常半年写不了一篇文章。为了能够督促自己将技术或是生活点滴记录下来，寻找一种使用Markdown语法，干净整洁的写作环境：github+jekyll，这正是我想要的。

## 升级Command Line
由于刚升级了"优圣美地"，需要安装一下Command Line Tools，安装jekyll需要。
```
$ xcode-select --install
```

## 更新RubyGems到最新版本，安装jekyll需要。
https://rubygems.org/pages/download

```
$ gem update --system          # may need to be administrator or root
$ gem install rubygems-update  # again, might need to be admin/root
$ update_rubygems              # ... here too
```

## 安装jekyll，纯静态blog，无需数据库支持；将markdown转为页面。
```
$ gem install jekyll           # may need to be administrator or root
```

## Clone jekyll-bootstrap，这是一个快捷搭建jekyll+github pages的工具。
```
$ git clone https://github.com/plusjade/jekyll-bootstrap.git
$ cd jekyll-bootstrap
$ jekyll serve
```

在Github上创建帐号，并创建一个USERNAME.github.io的仓库，这个就是你的个人主页的空间了。
然后将jekyll-bootstrap的目录结构导入到此仓库，jekyll的环境准备就绪。

## 替换jquery链接，由于默认theme里使用了googleapis的连接，导致访问会很慢或挂掉，所以这里替换为本地的jquery
下载一个jquery压缩版放入/assets/javascript/jquery-1.11.1.min.js

然后将_includes/themes/bootstrap-3/default.html中的
```
https://ajax.googleapis.com/ajax/libs/jquery/1.10.2/jquery.min.js
```
替换为：
```
/assets/javascript/jquery-1.11.1.min.js
```


## 中文支持
将jekyll默认的markdown解析引擎maruku 更换为RDiscount
```
sudo gem install rdiscount
```
然后在_config.yml中```highlighter: pygments```之后加一行```markdown: rdiscount```,这样再次生成网站的时候就使用RDiscount来解析md文件了。

## 个性化域名
USERNAME.github.io文件夹下新建一个CNAME文件，里面的内容就是你的域名，比如USERNAME.me

然后按照GitHub所说将你的域名指向204.232.175.78这个ip，这样当再次访问你的域名的时候，就会指向GitHub Pages了（可能DNS更新会需要一段时间）。

## 安装&更换theme
安装一个jekyll模板，然后使用它来变更blog界面。
```
rake theme:install git="https://github.com/jekyllbootstrap/theme-the-program.git"
rake theme:switch name="the-program"
```

## 调整首页
页面可以适当做一些调整，比如菜单名称等等。

## 开始写作...
```
rake post title="Hello World"
```
使用jekyll命令行创建文章，然后去_post目录下用markdown编辑器或vim编辑文章。