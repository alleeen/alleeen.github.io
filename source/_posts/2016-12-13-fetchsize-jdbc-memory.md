---
layout: post
title: Fetch Size 与 JDBC 内存管理
date: 2016-10-26 16:21:39
comments: true
ads: true
categories: [数据库]
tags: [JDBC,数据库,内存管理]
---

接触到 JDBC 的 Fetch Size 这个属性缘起一个性能问题，项目中需要将一个有千万级数据量的表中的记录导出到文件中去。按照正常的路数，先初始化连接；接着写好 SQL 语句，比如`SELECT * FROM DIM_USERS`；然后启动查询，拿到 ResultSet，最后遍历 ResultSet 将每行记录输出到文件中去。可在接下来的测试中，发现性能并不理想，在表中数据量小的时候，执行速度尚可接受，可是在进行大数据量压力测试的时候，发现代码往往要执行40分钟以上，这在实际生产环境上是万万不可接受的。

<!-- more -->

通过定位，发现性能瓶颈出现在从数据库中读取数据的时候，大概消耗了90%以上的时间。也就是说如果什么事情都不干，单纯对一个千万级数据量的 ResultSet 进行一次遍历就需要耗时35分钟以上。这样一来，问题就变得让人有点费解了，因为在同一套环境上，服务器向数据库写入数据的速率可以达到3万+/秒，为何查询变得如此低效？正在我百思不得其解的时候，一个大神走过来拍怕我的肩膀说：“小伙子，试试把 Fetch Size 调整一下。”

## Fetch Size

在 JDBC 中 Fetch Size 是 Statement 上的一个属性，先看下[Oracle 的帮助文档](https://docs.oracle.com/cd/E11882_01/java.112/e16548/resltset.htm#JJDBC28621)对它是怎么定义的：

>By default, when Oracle JDBC executes a query, it receives the result set 10 rows at a time from the database cursor. This is the default Oracle row-prefetch value. You can change the number of rows retrieved with each trip to the database cursor by changing the row-prefetch value

简单的说，这个属性控制了 JDBC 每次读取数据的行数，由于 JDBC 每次都要通过网络去读取数据，如果这个值配置得太小，那么就意味着在遍历 ResultSet 的时候 JDBC 需要频繁的通过网络读取数据，这就导致了读取数据时性能低下。那接下来的问题就简单了，就是将这个属性调大。可是调整到多少合适呢？1K、2K？还是1W、2W？要知道 JDBC 每次读取的数据是会缓存在内存中的，如果这个属性设置大了，就会使程序出现 OOM。

## JDBC Memory
接下来就得聊聊 JDBC 的内存管理了（这里特指 Oracle JDBC，别的厂商也许实现机制不是这样的）。JDBC 解析 SQL 语句后，为每个 Statement（包括 PreparedStatement 和 CallableStatement）分配了两个 Buffer 来缓存数据，`byte[]`和`char[]`。字符类型的数据（CHAR,
VARCHAR2, NCHAR, etc. ）缓存在`char[]`中，其他类型的数据缓存在`byte[]`中。在 SQL 语句解析后，语句所查询的列的数据类型就已经确定了，JDBC 会根据这些信息和 Fetch Size 一起计算出缓存的大小，并分配内存。所以如果不需要查询某张表的所以列时，使用`SELECT * FROM XXX`是一种浪费内存的行为，特别是表的列数多且数据量大的时候，很容易造成 OOM。

### 数据类型与内存占用
前面说了，JDBC 会根据查询语句中列的数据类型来计算缓存的大小那么每种数据类型大致占多少空间呢？请看下表。

| 数据类型         | 大小（byte）   | 备注   |
| -------------  |:-------------:|: -----: |
| VARCHAR2        |2             |每个字符占用2byte    |
| BFILE           |4K            |       |
| BLOB            | 4K           |       |
|CLOB             |4K            |       |
|Other            | 22           | 其他类型占用空间比较小，可以大致估算为22byte|

让我们来举个栗子：

```
CREATE TABLE TAB (ID NUMBER(10), NAME VARCHAR2(40), DOB DATE)
ResultSet r = stmt.executeQuery(“SELECT * FROM TAB”);
```

当 JDBC 解析查询语句时，数据库会告知 JDBC 结果会包含三列，NUMBER(10)、VARCHAR2(40) 和 DATE，第一列大概需要22 bytes，第二列包含了40个字符，所以需要`2 * 40`bytes，第三列也是大概需要22 bytes。因此，本次查询每条记录大致需要`22 + (40 * 2) + 22 = 124`bytes，如果 Fetch Size设置为10，那么缓存就需要分配1240 bytes 的空间。

## 如何正确设置Fetch Size
上面说了那么多无非就是想说明一个问题，就是 Fetch Size 的大小是要根据实际情况来设置，设置小了性能不好，设置大了内存会有问题。总之一个原则就是，在保证内存够用的情况下，尽量把 Fetch Size 设置得大一点。如果你拿不准设置多少，可以先试下下面的方式：

```
 4 * 1024 * 1024 / sum(所读取的列的数据长度)
```

## 参考文档：

+ [Oracle JDBC Memory Management](http://www.oracle.com/technetwork/database/enterprise-edition/memory.pdf)
