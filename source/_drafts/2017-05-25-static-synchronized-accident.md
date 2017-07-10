---
layout: drafts
title:  一起 Static 和 Synchronized 引发的血案
date: 2017-05-25 16:21:39
comments: true
ads: true
categories: [软件技术]
tags: [Java,多线程]
---

这两天在定位一个网上问题的时候发现一个很诡异的现象，系统夜间的汇总任务跑了很长一段时间才能结束，而且日志显示这些汇总任务的每个子任务都很快就结束了，但整体任务还是耗费了很长一段时间才结束。

```
// 日志
```

其实整体业务流程很简单，大致的流程就是系统创建了很多汇总任务，把它们丢到线程池中去执行。这些任务在执行的过程中，为了提高效率，会创建一些子任务并并发的运行它们，当子任务运行结束后，父任务就会结束，所以出现这种现象是非常不科学的。我的第一感觉就是是不是任务间存在不合理的锁竞争导致线程相互等待？仔细检查代码，果然发现了问题，在汇总任务类中有这样一个方法：

```java
private static synchronized format(DateTime dt){
  return "P" + dt.toString("yyyyMMHHmmss");
}
```