---
layout: post
title:  解决在 IntelliJ IDEA 时，搜狗输入法不跟随问题
date: 2017-08-06 16:21:39
comments: true
ads: true
categories: [软件技术]
tags: [Java,IntelliJ,搜狗输入法]
---

最近从华为离职并入职了新的公司，在新领的电脑配置好开发环境后就开始愉快的打码。可是在我要输入中文注释的时候，发现在 IDE 里面没法正常使用搜狗输入法，表现为输入法候选框不跟随光标，输入后不弹出候选字。

![输入法不跟随](/assets/images/2017-08-06-fix-sougou-intellij-compatibility/screen_print.png)

<!-- more -->

其实候选框不跟随光标还好，但无法弹出候选字确实没法忍，总不能不写注释或者全部用英文写注释吧。这么干的话，后面的维护者一定会有想砍死我的想法。尝试了重装或者升级输入法，均没有解决。这个版本的 IDEA 之前也用过，也没有碰到这个输入法的问题，仔细想了下配置的差异，之前我喜欢把 IDEA 自身使用的 JDK 设置为系统中已经安装的那一个，而这次为了图省事就没指定，那会不会是这个原因导致的？果然，切换后问题解决。

关于如何设置 IDEA 的 JDK 的问题，Jetbrains 有一份[官方文档](https://intellij-support.jetbrains.com/hc/en-us/articles/206544879-Selecting-the-JDK-version-the-IDE-will-run-under)可以供大家参考，我给大家简要说明一下：

+ 打开 IDEA 使用 `Help | Find Action`(可以使用快捷键 ：Ctrl+Shift+A 或者 Cmd+Shift+A Mac平台) ，输入`Switch IDE Boot JDK`，按下回车；
+ `Switch IDE Boot JDK dialog` 对话框会弹出来，在对话框中选择系统安装的 JDK 或者直接输入 JDK 的路径 (比如 c:\Program Files (x86)\Java\jdk1.8.0_112 或者 /Library/Java/JavaVirtualMachines/jdk1.8.0_112.jdk/Contents/Home/ or /usr/lib/jvm/open-jdk)。
+ 点击 OK，并重启 IDEA 客户端，重启后打开 About IntelliJ IDEA 看看 JDK 是否设置成功，见下图：

![About Page](/assets/images/2017-08-06-fix-sougou-intellij-compatibility/android-studio_jdk_8u131.png)
