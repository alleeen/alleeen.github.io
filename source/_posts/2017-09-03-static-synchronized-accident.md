---
layout: post
title:  一起 Static 和 Synchronized 引发的血案
date: 2017-09-03 16:21:39
comments: true
ads: true
categories: [软件技术]
tags: [Java,多线程]
---

这两天在定位一个网上问题的时候发现一个很诡异的现象，系统夜间的汇总任务跑了很长一段时间才能结束，而且日志显示这些汇总任务的每个子任务都很快就结束了，但整体任务还是耗费了很长一段时间才结束。

```
sub-job 1 done in 3s
sub-job 2 done in 3s
sub-job 3 done in 2s
sub-job 4 done in 5s
sub-job 5 done in 6s
sub-job 6 done in 8s
sub-job 7 done in 9s
...
whole process is down in 3235s
```

<!-- more -->

其实整体业务流程很简单，大致的流程就是系统创建了很多汇总任务，把它们丢到线程池中去执行。这些任务在执行的过程中，为了提高效率，会创建一些子任务并并发的运行它们，当子任务运行结束后，父任务就会结束，所以出现这种现象是非常不科学的。我的第一感觉就是是不是任务间存在不合理的锁竞争导致线程相互等待？仔细检查代码，果然发现了问题，在汇总任务的父类中有这样一个方法：

```java
private static synchronized format(DateTime dt){
  return "P" + dt.toString("yyyyMMHHmmss");
}
```

这个方法是汇总任务根据时间生成目标汇总时间周期用的，之所以会封装成一个方法，估计是为了代码复用考虑。封装本身并没有错，但是要命的是，开发人员将方法声明为`static synchronized`，让我们先回忆一下这个两个关键字的作用：

+ synchronized

  synchronized 关键字放在方法声明上时，表示该方法为`Synchronized Methods`，即同步方法，在[The Java™ Tutorials](https://docs.oracle.com/javase/tutorial/essential/concurrency/syncmeth.html)中对同步方法有以下描述：

  >First, it is not possible for two invocations of synchronized methods on the same object to interleave. When one thread is executing a synchronized method for an object, all other threads that invoke synchronized methods for the same object block (suspend execution) until the first thread is done with the object.

  >Second, when a synchronized method exits, it automatically establishes a happens-before relationship with any subsequent invocation of a synchronized method for the same object. This guarantees that changes to the state of the object are visible to all threads.

  简单来说就是当一个方法声明为同步方法的时候，不可能出现多个线程同时调用同一个对象（注意是同一个对象，这点很重要）上的该方法，只有当一个线程调用结束，其他线程才有可能获取锁并执行该方法。

+ static

  在Java中static表示“全局”或者“静态”的意思，用来修饰成员变量和成员方法，当然也可以修饰代码块。被static修饰的成员变量和成员方法是独立于该类的，它不依赖于某个特定的实例变量，也就是说它被该类的所有实例共享。所有实例的引用都指向同一个地方，任何一个实例对其的修改都会导致其他实例的变化。

那么`synchronized`加上`static`会出现什么效果？按照上面的分析`static`是整个类共享的，不仅仅是一个对象，那么`static synchronized`修饰的变量、方法或者代码段就是在类的粒度上进行同步，而不是仅仅是在对象粒度上。对于这个问题，[Java machine language specification](http://docs.oracle.com/javase/specs/jls/se7/html/jls-8.html#jls-8.4.3.6)中也有描述：

>For a class (static) method, the monitor associated with the Class object for the method's class is used.

>For an instance method, the monitor associated with this (the object for which the method was invoked) is used.

所以在我们的业务代码中，如果在父类中声明了一个`static synchronized`的方法，就意味着每个继承它的子类及其对象在调用这个方法时都会争夺这个锁，那么造成任务执行效率低下也就是必然的了。
