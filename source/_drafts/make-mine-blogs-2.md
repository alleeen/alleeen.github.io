---
layout: drafts
title: Allen's Blogs 创建历程（2）
date: 2016-09-18 22:09:10
comments: true
ads: true
categories: 随便乱写
tags: [Blogs,Jekyll,软件工程师的自我宣传]
---

## 从 Jekyll 到 Hexo

玩了一段时间的 Jekyll 后，发现其还是有一些不尽如人意的地方。也许正是因为“轻量级”，Jekyll 似乎并不能满足我*不折腾*的要求。接着我开始考察 Hexo，和Jekyll相比，选择Hexo主要原因是：

+ Jeky基于Ruby实现，安装Jeky需要搭建Ruby环境，在Windows搭建Ruby环境并不是被推荐的，而 Hexo基于NodeJs实现，在Windows上安装NodeJs开发环境简单。
+ Jekyll没有本地服务器，无法实现本地博文预览功能，需要上传到WEB容器中才能预览功能，而Hexo可以通过简单的命令实现本地的预览，并直接发布到WEB容器中实现同步。
+ 比较直接的另一个原因是在网上查找了很多博客的主题，发现Jekyll官网提供的主题都不怎么好看(可能是个人原因)，而Hexo的主题看的比较顺眼。
+ 两者都支持Markdown语法，这点我非常喜欢。
+ Hexo 有内置的 Deployer，可以很方便的部署到不同的站点上（为什么需要这个功能？到后面讲针对百度的 SEO 时，我会说到）。
