---
layout: post
title:  "由 Java 到 Scala：如何优雅的跳出循环"
date:   2016-09-16 19:27:03
comments: true
ads: true
categories: 软件技术
tags: [Java,Scala,跳出循环]
---

在开发过程中，我们经常会遇到这样的需求：循环执行某个操作，当满足一定条件的时候循环终止。最常见的场景就是累加数组中的元素，一直到大于某个值，用伪代码来描述就是：

```
DO LOOP{
  DO SOME THING
  IF SOME CONDITION
    BREAK
}
```
<!--more-->

如果我们使用 Java 来完成这样的需求，我们会这样完成我们的代码：

```java
// List[1,2,3,4,5,6]
int sum = 0;
for(int i = 0; i < list.size(); i ++){
  sum += list.get(i);
  if(sum > 4){
    break;
  }
}
```

在 Java 中，我们用一个`break`语句，就完成的从循环中跳出的工作。但在 Scala 中我们应该怎么做呢？要知道 Scala 特地没有在内置控制结构中包含 break 和 continue 是因为这两个控制结构和函数式编程有点格格不入。那么下面我将介绍几种在 Scala 中跳出循环的方法。

### 使用Return语句
没有`break`语句，那么作为 Java 的开发人员，第一时间就会想到`return`，还好 Scala 支持`return`

```scala
// List[1,2,3,4,5,6]
var sum = 0
list.foreach(i =>{
  sum += i
  if(sum > 4){
    return
  }
})
```

### 使用Breaks
在 Scala 2.8以上版本中，Scala 增加了`scala.util.control.Breaks`包，通过导入这个包，你可以在 Scala 中写出和 Java 中相似的带`break`语句的循环。

```scala
import scala.util.control.Breaks._
var sum = 0
breakable {
  for (i <- 0 to 6) {
   sum += i
   if (sum >= 4) break
 }
}
```

但是，这并不代表 Scala 从 2.8 版本开始支持`break`语句，它的实现实际是通过抛出异常给上级调用函数来达到控制循环的目的。`Breaks`的关键代码如下：

```scala
def breakable(op: => Unit) {
  try {
    op
  } catch {
    case ex: BreakControl =>
      if (ex ne breakException) throw ex
  }
}
```

所以，使用`Breaks`就等价于下面的代码：

```scala
object AllDone extends Exception { }
var sum = 0
try {
  for (i <- 0 to 6) { sum += i; if (sum>=4) throw AllDone }
} catch {
  case AllDone =>
}
```

## 一些优雅的方法
上面的方法虽然可以达到我们的目的，但和优雅还是差点距离，下面就回到我们的主题：如何优雅的跳出循环。

### 使用 Stream
Stream 是个很有意思的结构，它和列表相似，只不过它会延迟计算下一个元素，仅当需要的时候才会去计算。运用 Stream 的这个特性，我们可以用一种优雅的方式达到我们跳出循环的目的

```scala
var sum = 0
(0 to 6).toStream.takeWhile(_ => sum < 4).foreach(i => sum+=i)
```

你可能会觉得这个程序有 Bug，因为咋一看`takeWhile`中并没有进行累加，只比较了`sum < 4`，而累加是在`foreach`中做的，`takeWhile`的条件应该永远为`true`，导致最后的结果是错误的。那么到底会不会这样呢？答案是：不会。因为 Stream 是 Lazy 的，它会延迟计算下一个元素，在这个例子中，`takeWhile(_ => sum < 4)`只会在每次`foreach`需要取 Stream 中的一个元素出来累加的时候才会执行一次，这就保证了判断条件的有效性。大致的执行序列如下

```
// List[1,2,3,4,5,6]
var sum = 0
takeWhile(_ => 0 < 4)
foreach(1 => 0+=1)
var sum = 1
takeWhile(_ => 1 < 4)
foreach(2 => 1+=2)
....

```

### 使用递归代替循环
还有一种方法就是使用递归代替循环

```scala
var sum = 0
def addTo(i: Int, max: Int) {
  sum += i; if (sum < max) addTo(i+1,max)
}
addTo(0,6)
```
