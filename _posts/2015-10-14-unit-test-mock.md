---
layout: post
title:  "Mock 与 Stub"
date:   2015-10-14 16:50:39
comments: true
categories: 软件技术
tags: [Java, 单元测试]
---

>在进行单元测试的时候，我们会发现我们要测试的方法会有很多外部依赖，比如：发邮件，进行网络通讯，操作文件系统等等。而我们通常关注的是被测试对象的功能和行为，对于它的依赖，我们仅仅需要关注它们之间的交互，但对依赖的对象是如何执行的具体细节我们并不关注。较为常见的技巧就是使用mock对象或者stub对象来代替真实的依赖。

##Mocks aren't stubs

这是软件大师[Martin Fowler](http://martinfowler.com/)的一篇经典博文。Martin大师在文章中详细的解释了Mock与Stub的区别，以及怎样使用它们进行TDD实践等等一系列干货，强烈推荐阅读，猛击[这里](http://martinfowler.com/articles/mocksArentStubs.html)阅读原文。我无意把大师的话再复述一遍，所以在本文中我就聊聊我对Mock与Stub的理解以及一些实践。

<!--more-->

###相同点

先看看两者的相同点吧，非常明确的是，Mock和Stub都可以用来对系统(或者将粒度放小为模块，单元)进行隔离。先看看两者的相同点吧，非常明确的是，Mock和Stub都可以用来对系统(或者将粒度放小为模块，单元)进行隔离。

###不同点

Mock和Stub有两个主要区别：

1. 校验测试结果的方式不同，Mock倾向于校验行为（Beahavior verification），Stub倾向于校验状态；
2. Mock和Stub也代表了两种将测试与设计结合在一起的理念。

上面的说法比较抽象，让我们通过例子来看看Mock与Stub的区别。

##使用Stub进行单元测试

下面是一个使用Stub进行单元测试的例子，我们打算创建一个订单对象，并用仓库中的货物填充这个订单。这个订单对象很简单，只有产品和数量两种信息，仓库保存着不同产品的目录。当我们需要填充订单的时候，会有两种不同的回应，如果仓库中有足够的货物，那么订单就会被填满，并且仓库相应产品的数量就会降低到对应的数量。如果仓库中没有足够的参评，那么订单就不会被填充，并且仓库中产品的数量没有任何的变化。

```java
public class OrderStateTester extends TestCase{
    private static String TALISKER = "Talisker";
    private static String HIGHLAND_PARK = "Highland Park";
    private WareHouse warehouse = new WareHouseImpl();

    protected void setup() throws Exception{
        warehouse.add(TALISKER , 50);
        warehouse.add(HIGHLAND_PARK , 25);
    }

    public void testOrderIsFilledIfEnoughInWarehouse(){
        Order order = new Order(TALISKER , 50);
        order.fill(warehouse);
        assertTrue(order.isFilled());
        assertEquals(0 , warehouse.getInventory(TALISKER));
    }

    public void testOrderDoseNotRemoveIfNotEnough() {
        Order order = new Order(TALISKER , 51);
        order.fill(warehouse);
        assertFalse(order.isFilled());
        assertEquals(50 , warehouse.getInventory(TALISKER));
    }
}
```

上面的例子里，我们需要对Order对象进行测试，为了验证Order.fill方法，我们还需要WareHouse对象。但真正的WareHouse对象内部可能有很复杂的实现，比如读取文件，访问数据库，持有同步锁以维持对象在并发访问时内部数据正确等。实际上在单元测试时我们并不需要去和这些代码发生交互，而且这些复杂的代码还会让我们的单元测试很不稳定。数据库连接失败、必须的配置文件读取失败等都会导致我们的单元测试失败。显然我们并不希望这些外部的因素影响我们的单元测试
