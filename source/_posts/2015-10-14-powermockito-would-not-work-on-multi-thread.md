---
layout: post
title:  "在多线程构建场景下Powermockito无法在不同类中Mock同一个静态方法"
date:   2015-10-14 13:50:39
comments: true
ads: true
categories: 软件技术
tags:
  - Java
	- 单元测试
---

在修改单元测试的过程中，不幸踩了个坑，发现 Powermockito 的PowerMock.mockStatic(ClassThatContainsStaticMethod.class) 在多线程场景下是无法正常工作的，这再次验证了之前 ThrougthWorks 顾问说的那句话：

>除非万不得已，或者是Mock遗留系统接口，否则不要使用Powermockito。

<!--more-->

发生问题的场景是这样的 Class C 有一个静态方法，Class A 和 Class B 都需要调用这个方法完成一些功能：

```java
Class C{
	public static SomeObject getSomeObject(){
		[....]
	}
}

Class A {
	private SomeObject someObject = C.getSomeObject();

	[.....]
}

Class B {
	private SomeObject someObject = C.getSomeObject();

	[.....]
}
```

由于在测试中直接调用 C.getSomeObject() 会导致一些不可预期的错误，所以我想对AB类进行测试就必须使用Mock，于是我那么写：

```java
Class ATest{
	@Before
	public void setUp(){
		PowerMock.mockStatic(C.class)
		PowerMock.when(C.C.getSomeObject()).thenReturn(PowerMock.mock(SomeObject.class))
	}

}

Class BTest{
	@Before
	public void setUp(){
		PowerMock.mockStatic(C.class)
		PowerMock.when(C.C.getSomeObject()).thenReturn(PowerMock.mock(SomeObject.class))
	}

}

```

当我在IDE中分别运行 ATest 或者 BTest 是，我的测试都是能正确运行的，但是当你使用Maven或者其他的构建工具进行多线程测试的时候，你就会发现问题来了。一会是A抛异常，一会是B抛异常，总之就是不能很好的工作。由于我不是Powermockito的专家，所以无法深入的去探究这个问题的原因，但是我想，这应该是和静态方法本身在一个JVM内的唯一性有关，我截取了网上两个解释供参考：

#### Explanation 1

Without going into details let's look at this code written using with Mockito :

```java
given(mock.doSomethingWith(eq("A"), longThat(...)).thenReturn("C");
```

Which is roughly equivalent to :
(***** NEVER use a reference to OngoingStubbing in real test code, it might >lead to wrong test code *****)

```java
String aString = eq("A");
Long aLong = longThat(...);
String variableThatGiveReturnType = mock.doSomethingWith(aString, aLong);
BDDOngoingStubbing<String> ongoingStubbing = given(variableThatGiveReturnType);
ongoingStubbing.thenReturn("C");
```

The stubbing is clearly not finished until the last call thenReturn is completed, right.

Don't you see the missing link between all those line to actually achieve the stubbing in a fluent way ? ;)

Dependening on how you do that, if you don't synchronize this block you won't be able to achieve any correct stubbing, otherwise concurrent access anywhere in this block will garble things in the mock internals.

And if you add the fact that the mock might be already used, with it's own concurrent code to use the answers, you end up in with completely messed up internal states.

Anyway, always stub before using mocks concurrently.

#### Explanation 2

For healthy scenarios Mockito plays nicely with threads. For instance, you can run tests in parallel to speed up the build. Also, You can let multiple threads call methods on a shared mock to test in concurrent conditions. Check out a [http://mockito.googlecode.com/svn/tags/latest/javadoc/org/mockito/Mockito.html#22 timeout()] feature for testing concurrency.

However Mockito is only thread-safe in healthy tests, that is tests without multiple threads stubbing/verifying a shared mock. Stubbing or verification of a shared mock from different threads is NOT the proper way of testing because it will always lead to intermittent behavior. In general mutable state + assertions in multi-threaded environment lead to random results. If you do stub/verify a shared mock across threads you will face occasional exceptions like: WrongTypeOfReturnValue, etc.
