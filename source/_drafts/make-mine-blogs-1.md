---
layout: drafts
title:  "Allen's Blogs 创建历程（1）"
date:   2016-08-30 16:21:39
comments: true
ads: true
categories: 随便乱写
tags: [Blogs,Jekyll,软件工程师的自我宣传]
---

很早很早以前我就开始玩博客，陆陆续续注册了很多平台，比如博客中国、cnblogs、javeeye（现在叫iteye）、csdn，也零零散散写了一些文章，不过没有坚持多久，工作忙起来后就不再更新，自我回顾一下好像还真没有什么干货，只算是给互联网里堆了一串01010101的数据罢了。那为什么最近又动了写 Blogs 的心思，原因是最近读了一本书，书名是：[《软技能：代码之外的生存指南》](http://product.china-pub.com/4971248)，里面*第二篇：自我营销*中讲到程序员自我营销的重要性，其中一点就提到了写 Blogs。总结来说，程序员写写 Blogs 不仅是自我营销的一种方式，还是一种很好的学习方式，不是说知识能说出来才算学到了么。

<!-- more -->

### Jekyll & GitHub Pages
自我总结一下，之前没有坚持下来很大一个原因就是一个字：“懒”，再加上之前的那些 Blogs 系统多多少少会有点不足。我也曾经尝试过购买 VPS 主机，自己搭建 Blogs，我甚至还为之购买了域名，可是后来发现，为何 VPS 是何等的费时费力。要安装软件，要安装数据库，还要防止被盗链导致流量不够用，哎，都是泪，不说了。到最近，无意中看到一篇译文，似乎是[《像黑客一样写博客》](http://tom.preston-werner.com/2008/11/17/blogging-like-a-hacker.html)，瞬间就被带上车了，开始使用 Jekyll 和 GitHub Pages 架设我的静态博客。

要在使用 GitHub Pages 服务，首先需要创建一个名字叫 “[你的用户名].github.io” 的项目：

![图片来自：GitHub](/assets/images/make-mine-blogs-1/uset-repo@2x.png)

接着把新建好的项目 Clone 下来，有两种方式 Clone 项目，一种是点击项目右上角的绿色“Set up in desktop”按钮使用 Github 客户端 Clone 项目；另外一种就是通过终端命令行来 Clone 项目。

```sh
$ git clone [you project addr] [your locale dir]
```

Clone 完成后，需要在本地搭建 Jekyll 的写作环境，正式开启静态博客之旅。

#### Jekyll 环境准备
首先安装必要工具

- Ruby：Mac OS X 10.5以上都自带
- RubyGems：Mac OS X 10.5以上都自带
- Xcode Command-Line Tools： 安装Xcode会自动安装，检查Preferences → Downloads → Components是否有Command-Line Tools这项提供下载，如果没有说明已安装
- git：命令行输入git --version检查是否已安装，下载地址：[http://sourceforge.net/projects/git-osx-installer/](http://sourceforge.net/projects/git-osx-installer/)

在国内 gem 源地址可能已经被墙（万恶的 GFW），所以你可能需要将 gem 源替换为淘宝的镜像源：

```sh
// 移除官方镜像源
$ gem sources --remove https://rubygems.org/
// 添加淘宝镜像源，或者其他镜像地址
$ gem sources -a http://ruby.taobao.org/
// 验证是否替换成功
$ gem sources -l
```

如果终端中出现下面的显示则代表替换成功。

```sh
*** CURRENT SOURCES ***
http://ruby.taobao.org/
```

接着开始安装 Jekyll

```sh
// 更新下 gem
sudo gem update --system
```
MAC 系统版本如果是 El Capitan 使用下面这个命令。这是因为 Apple 在 OS X El Capitan 中全面启用了名为 System Integrity Protection (SIP) 的系统完整性保护技术。受此影响，大部分系统文件即使在 root 用户下也无法直接进行修改，所以需要把安装路径替换为用户有写入权限的目录。

```sh
sudo gem update -n /usr/local/bin --system
```

如果你嫌每次都要打安装路径比较麻烦，你也可以把它变成默认配置，在用户根目录下创建一个名为`.gemrc`的文件，在里面写入`gem: -n/usr/local/bin`，并保存。或者使用下面的命令：

```
echo "gem: -n/usr/local/bin" >> ~/.gemrc
```

接下来安装 Jekyll

```sh
$ sudo gem install jekyll
// 如果提示权限错误，请使用下面的命令
$ sudo gem install jekyll -n /usr/local/bin
```

OK，这样 Jekyll 环境就安装完成了，接下来导入 Jekyll 后，就可以开始写作了。在网络上有很多漂亮的 Jekyll 主题可供你选择，你可以访问[jekyllthemes.io](http://jekyllthemes.io/)找到你喜欢的主题并下载下来，或者通过 Google 搜索，如果还不满意，你也可以选择自己创建一个主题。选择好你喜欢的 Jekyll 的主题后，将主题复制到前面从 Github 上 Clone 的项目文件夹中去。一个典型的 Jekyll Blogs 的目录结构应该如下面所示：

```
.
├── _config.yml
├── _drafts
|   ├── begin-with-the-crazy-ideas.textile
|   └── on-simplicity-in-technology.markdown
├── _includes
|   ├── footer.html
|   └── header.html
├── _layouts
|   ├── default.html
|   └── post.html
├── _posts
|   ├── 2007-10-29-why-every-programmer-should-play-nethack.textile
|   └── 2009-04-26-barcamp-boston-4-roundup.textile
├── _data
|   └── members.yml
├── _site
├── .jekyll-metadata
└── index.html
```

在该目录下执行：

```sh
$ jekyll server // 简写 jekyll s
```

在浏览器地址栏中输入：http://localhost:4000/ 就可以看到刚才新建的 Blog 长什么样子了。在这里新增、修改、删除文章都可以实时的看到，只需要刷新页面即可。你可以试着修改那篇默认文章看看效果。
