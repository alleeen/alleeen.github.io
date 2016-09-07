---
layout: post
title:  "为什么java.util.concurrent 包里没有并发的ArrayList实现？"
date:   2016-09-07 19:27:03
comments: true
ads: true
categories: 软件技术
tags: [Java,多线程,并发]
---

问：JDK 5在 java.util.concurrent 里引入了 ConcurrentHashMap，在需要支持高并发的场景，我们可以使用它代替 HashMap。但是为什么没有 ArrayList 的并发实现呢？难道在多线程场景下我们只有 Vector 这一种线程安全的数组实现可以选择么？为什么在 java.util.concurrent 没有一个类可以代替 Vector 呢？

答：我认为在 java.util.concurrent 包中没有加入并发的 ArrayList 实现的主要原因是：很难去开发一个通用并且没有并发瓶颈的线程安全的 List。像 ConcurrentHashMap 这样的类的真正价值（The real point / value of classes）并不是它们保证了线程安全。而在于它们在保证线程安全的同时不存在并发瓶颈。举个例子，ConcurrentHashMap 采用了锁分段技术和弱一致性的Map迭代器去规避并发瓶颈。所以问题在于，像“Array List”这样的数据结构，你不知道如何去规避并发的瓶颈。拿contains() 这样一个操作来说，当你进行搜索的时候如何避免锁住整个 list？另一方面，Queue 和 Deque (基于Linked List)有并发的实现是因为他们的接口相比List的接口有更多的限制，这些限制使得实现并发成为可能。CopyOnWriteArrayList 是一个有趣的例子，它规避了只读操作（如 get/contains）并发的瓶颈，但是它为了做到这点，在修改操作中做了很多工作和修改可见性规则。 此外，修改操作还会锁住整个List，因此这也是一个并发瓶颈。所以从理论上来说，CopyOnWriteArrayList 并不算是一个通用的并发 List。
