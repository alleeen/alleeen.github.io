---
layout: post
title:  "Mac OS X 安装 JDK备忘"
date:   2013-09-30 16:21:39
comments: true
ads: true
categories: 日积月累
tags: [JDK,Macbook]
---

### 安装JDK1.6
oracle官网从jdk1.7开始才有Mac版的安装包，但有的项目必须使用jdk1.6，所以必须从其他途径安装jdk1.6了。查了下发现，要想安装jdk1.6，可以直接从apple的开发者网站下安装提供的java支持包，具体下载地址 [http://connect.apple.com/](http://connect.apple.com/)

详细可参见这篇文章
[http://stackoverflow.com/questions/6614380/jdk-on-osx-10-7-lion](http://stackoverflow.com/questions/6614380/jdk-on-osx-10-7-lion)

### 包路径等问题

系统默认安装的JRE路径`/System/Library/Frameworks/JavaVM.framework/`，oracle和apple等安装的JDK包的路径`/Library/Java/JavaVirtualMachines/`

### JAVA_HOME在哪了？
```
/Library/Java/JavaVirtualMachines/1.6.0_38-b04-436.jdk/Contents/Home
```

注：1.6.0_38-b04-436.jdk目录名字与安装的jdk版本有关

### rt.jar、jsse.jar去哪了？
rt.jar已经集成到`/Library/Java/JavaVirtualMachines/1.6.0_38-b04-436.jdk/Contents/Classes/classes.jar`，jsse.jar也在Classes目录下

建议把`classes.jar`和`jsse.jar`建立软连接到`/Library/Java/JavaVirtualMachines/1.6.0_38-b04-436.jdk/Contents/Home/lib/`下，并且`classes.jar`的软连接命名为`rt.jar`

这样就可以避免一些时候会发生找不到`rt.jar`的问题了，例如在使用混淆码的时候。

### 配置JAVA_HOME
Mac OS X的环境变量文件在`/etc/profile`，unix一贯重要的文件。
在此添加最下端添加

```
JAVA_HOME=/Library/Java/JavaVirtualMachines/1.6.0_38-b04-436.jdk/Contents/Home/
export JAVA_HOME
```
