---
layout: post
title:  "解决mac升级10.11后，出现的 xcrun: error: invalid active developer path, missing xcrun 错误"
date:   2016-09-24 13:50:39
comments: true
ads: true
categories: 软件技术
tags: [Mac OS X, git, 升级]
---

前天把小mac升级到了10.11，结果今天在终端里使用git的时候，弹出一行莫名其妙的错误：`xcrun: error: invalid active developer path (/Library/Developer/CommandLineTools), missing xcrun at: /Library/Developer/CommandLineTools/usr/bin/xcrun`。去google了一圈，找到了一个github上homebrew issues里很老的帖子，按着里面说的，重装了一下xcode command line，结果就正常了……

```
xcode-select --install
```

不过看帖子里并不是所有人重装都能解决问题，有些人似乎还要手动切换下xcode的路径才能解决。

```
sudo xcode-select -switch /
```

因为帖子标题说是在升级到“冲浪湾”时遇到了这问题，所以看来这问题属于每次升级时候都会碰到的月经型问题了OTL。问题解决后，我又去各处翻了下问题出现的原因，可惜没有找到。个人推断可能是因为git所需的lib关联到了command line tools，升级时改动了lib的路径所致吧。
