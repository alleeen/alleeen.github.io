---
layout: post
title:  "在公司内网如何更新IntelliJ的插件"
date:   2016-08-19 22:21:39
comments: true
ads: true
categories: 软件技术
tags: [Java, IntelliJ,ssl]
---

最近小伙伴们更新IntelliJ后，发现没法安装或者更新插件了，每次尝试在线安装时总会提示SSL错误。特别是要玩Scala的小伙伴更是抓狂，因为本身IntelliJ并不自带Scala的支持，需要下载Scala插件。不得以，只能通过手动下载，但是这样就不能享受插件更新的新功能了，很是不爽。那么报SSL错误的原因是什么呢？其实是因为IntelliJ更新插件时使用了Https连接，在连接时，客户端和服务器是要相互校验证书的，一般来说，只要证书正确，客户端是可以和服务器正常交互的。但是，我们是在公司内网，用的是公司的Proxy连接外网。公司的代理服务器会将证书换成公司自己颁(wei)发(zao)的证书（满满的[中间人攻击](https://zh.wikipedia.org/wiki/%E4%B8%AD%E9%97%B4%E4%BA%BA%E6%94%BB%E5%87%BB)的即视感，公司这样做是要干嘛？你懂的。。。），这时IntelliJ就无法同插件服务器正常通信了，那么怎么解决这个问题呢？那就是导入公司代理服务器的根证书，把公司颁(wei)发(zao)的证书变成可信任的证书。

<!--more-->

OK, Let's do it! 首先导出公司代理服务器的根证书，用浏览器即可，随便访问应该https的外网网站，点击地址栏上的小锁头。

![ie_url_bar](/img/java-ssl-error/ie_url_bar-certificate-error.png)

打开的窗口中,点击下一步即可,

![证书详细信息](/img/java-ssl-error/export_cert_1.png)

在正式编码格式中,选择指定的格式,点击下一步;

![证书导出向导](/img/java-ssl-error/export_cert_2.png)

指定生成证书文件的名称(此处为vbooking.cer)

![vbooking.cer](/img/java-ssl-error/export_cert_3.png)

接着，将证书导入java的cacerts证书库，切换到目录 ${JAVA_HOME}/jre/lib/security, 执行如下命令

```
keytool -import -alias vbooking -keystore cacerts -file ${cert_file_path}  
```

其中：

* -alias 指定别名(推荐和证书同名)
* -keystore 指定存储文件(此处固定)
* -file 指定证书文件全路径(证书文件所在的目录)

此时命令行会提示你输入cacerts证书库的密码,敲入changeit即可,这是java中cacerts证书库的默认密码,当然也可自行修改。

最后，在系统中新建一个环境变量，IDEA_JDK（64位程序为IDEA_JDK_64），指向刚才导入根证书的JDK，不然IntelliJ会使用内置的JDK（详细见[这里](https://intellij-support.jetbrains.com/hc/en-us/articles/206544879-Selecting-the-JDK-version-the-IDE-will-run-under)），重启IntelliJ后即可。
