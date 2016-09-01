---
layout: post
title:  "如何在单元测试中设置系统环境变量"
date:   2015-10-17 13:50:39
comments: true
ads: true
categories: 软件技术
tags: [Java, 单元测试]
---

有时我们需要通过读取系统环境变量来获取一些有用的信息，比如系统路径、临时目录等。在系统真正运行的时候我们可以通过启动命令行，如：java -Dxxx.xxx=xxxx ...，或者使用System.setProperty("xxx.xxx", "xxx.xxx")来设置系统环境变量。但在单元测试时如何设置这些系统环境变量又成了一个让人头疼的问题。有些小伙伴是在setUp方法里设置，比如：

```java
    @Before
    public void setUp() throws LicenseException
    {
        PowerMockito.mockStatic(XXXSystem.class);
        System.setProperty("xxx.xxx", "xxx.xxx");
    }
```

<!-- more -->

但是我们很快就会发现，这种设置方法在只有一个测试用例的时候是OK的，当你的测试类里有多个@Test标签时，就会发生一些很奇怪的问题。比如某些用例读到了环境变量，有些却没有读取到。主要的原因是System.setProperty("xxx.xxx", "xxx.xxx");方法是会作用在整个JVM上的，而多个测试用例是会在同一个JVM上面运行的，而JUnit的@Before标签标示的方法又会在每个测试用例启动前运行，这样就会导致环境变量相互覆盖。特别是开启并发执行单元测试功能时，这种现象更加严重。那么如何设置环境变量才是安全的呢？首先，我们要抛弃在setUp方法里设置环境变量的做法，然后在POM文件中做如下配置：

```xml
<properties>
    <!-- 单元测试时，系统参数iemp.home的路径-->
    <test.home>${project.build.directory}/opt/server</test.home>
</properties>
    ...
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>2.18.1</version>
    <configuration>
        ....
        <systemPropertyVariables>
            <home>${test.home}</home>
        </systemPropertyVariables>
    </configuration>
</plugin>

```

这样我们就可以很轻松的在单元测试中读取系统环境变量了。
