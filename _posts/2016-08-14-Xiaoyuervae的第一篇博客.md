---
layout: post
title: "使用Jekyll快速搭建你的博客系统"
date: 2016-08-14 
description: "xiaoyuervae的博客"
tag: 博客 
---

现在我要介绍一下本站的建立.也是我在使用`Jekyll`的时候的一些记录

<!--more-->

### 写在前面
> 什么是`Jekyll` ?

jekyll是一个简单的免费的Blog生成工具，类似WordPress。但是和WordPress又有很大的不同，原因是jekyll只是一个生成静态网页的工具，不需要数据库支持。但是可以配合第三方服务,例如Disqus。最关键的是jekyll可以免费部署在Github上，而且可以绑定自己的域名。

> 我为什么选择使用`Jekyll` 建立自己的博客 ?

一直想建立自己的博客写一些自己的学习记录,但自己真的是有点懒,之前是想在阿里云上部署自己的博客,但是面临最大的问题是自己对于设计方面略显疲惫啊~~~~(>_<)~~~~,对于之前一直写Java的我来说简直是有点残忍。

某日看到群里有人使用了HEXO建站,觉得好奇便在度娘那里询问了一波,之后偶然发现了[潘柏信](https://baixin.io)同学在简书上面的这篇文章[HEXO+Github,搭建属于自己的博客](http://www.jianshu.com/p/465830080ea9),但是他现在的个人主页已经不是用的hexo了,开始用hexo建了博客但是老感觉主题不大好看,于是`顺藤摸瓜`的找到了[`猫神`](https://onevcat.com)的[vno-jekyll](https://github.com/onevcat/vno-jekyll) 主题,真心觉得这个主题好看并且简洁。在此再次感谢二位的铺路。

### 配置安装环境

下面是快速起一个Jekyll模板的代码：

```zsh
~ $ gem install jekyll
~ $ jekyll new myblog
~ $ cd myblog
~/myblog $ jekyll serve
# => Now browse to http://localhost:4000
```
## 这是在你的系统为mac os的情况下:
在使用gem之前你需要先安装[ruby](http://www.ruby-lang.org/en/downloads)另外还有[rubygem](https://docs.rubygems.org/)
安装完成之后使用jekyll serve 应该是这样的：
![1.png](/assets/images/2016-08-14/1.png)

接下来你可以在github上面创建你自己的博客仓库啦！

### 在github上创建你自己的repository

github是学习的利器，大家的代码基本上都托管在上面，注册过程和普通的社区注册差不多，这里我也就不赘述了。创建自己的仓库名为yourname.github.io(*ps:yourname就是你的账户名。比如说我的就是xiaoyuervae.github.io*)


接下来你就可以到bash或者zsh中clone我仓库中的代码到你的本地仓库了,
*执行以下代码：*

```bash
cd /volumes/docformac/git/xiaoyuervae.github.io/
git clone https://github.com/xiaoyuervae/xiaoyuervae.github.io
```

这时候你的目录结构应该像下面这样：

![2.png](/assets/images/2016-08-14/2.png)


### 运行Jekyll博客

>* bundler exec jekyll serve

执行上面这行代码打开浏览器输入`localhost:4000`:
![3.png](/assets/images/2016-08-14/3.png)

### 更改_config.yml配置
![4.png](/assets/images/2016-08-14/4.png)
![5.png](/assets/images/2016-08-14/5.png)

里面的配置读起来也很简单，把里面的信息改成自己的就可以了。

### 使用多说评论系统
![6.png](/assets/images/2016-08-14/6.png)

当当当，就是这个非常好用的多说评论系统，上[duoshuo](http://duoshuo.com/) 注册一个填好自己的域名然后就可以使用了。
评论是存在他们的服务器的，具体想了解一下的可以上官网细细了解。

### diy你的博客
![7.png](/assets/images/2016-08-14/7.png)

**上面的图片给大家示例了一下html文件中的引用，按照上面的规则我们可以在js或者css文件夹下自行修改成自己喜欢的样式**

另外大家也可以新增或者修改.html文件加上可以提现出自己特色的组件。


***

今天就写到这里，之后有想到的再更新。现在我的博客也才刚刚搭建起来，还有许多东西需要完善，现在的ui还是用的几位大神的，再次感谢[潘柏信](https://baixin.io)还有[`猫神`](https://onevcat.com)。

以上。






















