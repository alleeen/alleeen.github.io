---
layout: post
title:  "玩花招的PowerMock"
date:   2016-09-01 23:31:03
comments: true
ads: true
categories: 转载
tags: [Java,单元测试，PowerMock]
---

本文转载至：[逸言](http://agiledon.github.io/blog/2013/11/21/play-trick-with-powermock/)，感谢原作者的精彩分享

当我们面对一个遗留系统时，常见的问题是没有测试。正如Michael Feathers在Working Effectively with Legacy Code一书中对“遗留代码”的定义。他将其简单归纳为“没有测试的代码”。真是太贴切了！正是因为没有测试，使得我们对遗留代码的任何重构都有些战战兢兢，甚至成为开发人员抵制重构的借口。从收益与成本的比例来看，对于这样的系统，我一贯认为不要盲目进行重构。因为重构的真正适用场景其实是发生在开发期间，而非维护期间。当然，提升自己的重构能力，尤其学会运用IDE提供的自动重构工具，可以在一定程度上保障重构的质量。然而，安全的做法，还是需要为其编写测试。

<!--more-->

测试是分层的，即使是针对自动化测试。面对遗留系统，成本相对较低的是针对功能特性编写的功能测试（或者说是验收测试），这可以运用一些BDD框架如Cucumber、JBehave等。由于它的测试粒度较粗，可以以较少的测试用例覆盖系统的主要功能。然而，它的缺点同样存在，那就是反馈周期相对较长。这就好像你置身一个陌生的城市，在找不到路的情况下，只是跟着感觉走。走了数十公里之后，方才幡然醒悟，想起要翻一翻带在手上的地图。倘若发现方向走错，再要回转就已经晚了。反馈周期最短的自然是单元测试。同样根据Michael Feather的定义，单元测试一定要快，一定要不依赖于外部资源。单元测试的粒度自然是最小的，但不要直观地认为单元测试就是针对方法。若只是针对方法来编写单元测试，就会陷入为测试而测试的怪圈。即使是位于技术象限的单元测试，我们仍然要按照业务规则来编写。一个测试方法应该对应一个粒度最小的原子功能。

要让单元测试跑得快，还要不吃草（依赖外部资源），应该怎么办？答案呼之欲出，那就是Mock。Mock当然不是万能的，记得胡凯写过一篇文章，提及Mock不是银弹。我知道他仅仅是为了强调这个观点，避免太多人过于依赖Mock，因为Brooks早就发表过论断，在软件行业，其实根本就“没有银弹”。关于Mock的争论由来已久，对此，我准备避而不谈。至少在我看来，如下几点基本已成定论：

1、是Mock行为，而非Mock数据；如果是针对数据，则应该属于Stub的范畴；

2、Mock通常发生在三种情况（让我们假设被测试对象为消费者，它要协作的对象为服务，此时需要Mock服务）：服务的行为只有定义，还未实现；服务需要访问外部资源（这意味着它可能很慢，也意味着它需要依赖外部资源）；服务的行为结果不确定（例如天气服务，股票服务）。

自然，我们不需要自己写Mock，有许多现成的好用框架，例如Java平台下的Mockito与EasyMock，.NET平台下的Moq，以及C++下的Google Mock和MockCpp。

然而，问题依然存在。考虑这样两种情况：

1、当我们要Mock的服务，其实是Utils的静态方法时，应该怎么办？

2、当我们要测试的方法内部直接实例化了协作的服务对象，又该怎么办？

显然，这是设计和代码的坏味道，它明显违背了DIP原则，即它不应该依赖于细节，而应该依赖于抽象。换言之，它产生了对服务对象的具体依赖。若要遵循DIP，就应该在被测对象的外部来注入依赖。这种紧耦合酿成了我们设计的类不具备良好的可测试性。

一个蠢蠢欲动的声音在说：让我们重构吧！且住，先让我们把这苛求的眼光放柔和一点。当你视所有丑陋的代码为“蝼蚁”时，那是因为你站在了足够的高度。可是站得太高，往往摔得更惨。现在，还是脚踏实地，先设身处地地考虑这样的场景：这是一个代码行数超过1000万行的软件系统，一共有十余个开发团队，一百多名开发人员在这个团队中工作。这个系统几乎没有测试，而系统的Jar包则达到上千个。这些Utils的静态方法被数十乃至上百个类调用，牵涉到的模块也有多个甚至十余个。而且，这个系统并没有引入任何一个IoC容器。有了这样一个背景，让我们再把柔和的眼光变得锐利一点，分析分析重构的可行性。要消除前面提到的坏味道，就需要将这些静态方法修改为实例方法，并通过依赖注入的方式注入。这个变化带来的是对整个系统的全局影响，即使我们有一些自动化重构的手段，仍然不认为这种重构一定就是可行的。

这就是我要谈PowerMock的前提！

现在，轮到玩花招的PowerMock出场了。有了它，什么静态方法，方法内部实例，乃至私有方法，统统都是浮云。而且，它对Mockito与EasyMock的扩展，使得我们更容易熟悉它的语法。要使用它很简单，需先设置对它的依赖。我选择了PowerMock针对Mockito的扩展：

```xml
        <dependency>
            <groupId>org.powermock</groupId>
            <artifactId>powermock-api-mockito</artifactId>
            <version>1.5.1</version>
        </dependency>
        <dependency>
            <groupId>org.powermock</groupId>
            <artifactId>powermock-module-junit4</artifactId>
            <version>1.5.1</version>
        </dependency>
        <dependency>
            <groupId>org.mockito</groupId>
            <artifactId>mockito-all</artifactId>
            <version>1.9.5</version>
        </dependency>
```

让我先给出如下的一份奇奇怪怪的设计，它主要是为了迎合之前提到的代码臭味。

```java
public class EmployeeTableUtil {
    public int count() {
        return 0;
    }

    public static final List<Employee> findAll() {
        return new ArrayList<Employee>();
    }

    public void insert(Employee employee) {
        if (existed(employee.getId())) {
            throw new ExistedEmployeeException();
        }

        //insert employee
    }

    public static final void update(Employee employee) {
        if (employee == null) {
            throw new NullEmployeeException();
        }
    }

    public boolean delete(Employee employee) {
        if (existed(employee.getId())) {
            //delete employee
            return true;
        }
        return false;
    }

    private boolean existed(String id) {
        return false;
    }
}

public class EmployeeRepository {
    private EmployeeTableUtil tableUtil;

    public int count() {
        return new EmployeeTableUtil().count();
    }

    public List<Employee> findAll() {
        return EmployeeTableUtil.findAll();
    }

    public boolean insert(Employee employee) {
        try {
            tableUtil.insert(employee);
            return true;
        } catch (ExistedEmployeeException e) {
            return false;
        }
    }

    public boolean update(Employee employee) {
        try {
            EmployeeTableUtil.update(employee);
            return true;
        } catch (NullEmployeeException e) {
            return false;
        }
    }

    public boolean delete(Employee employee) {
        return tableUtil.delete(employee);
    }

    private double bonus(Employee employee) {
        return employee.getSalary() * 0.1d;
    }

    public void setTableUtil(EmployeeTableUtil tableUtil) {
        this.tableUtil = tableUtil;
    }
}
```

现在，我要针对EmployeeRepository编写测试，它协作的服务类为EmployeTableUtil，主要承担了访问数据库的职责。在测试EmployeeRepository时，我们需要去Mock协作对象EmployeeTableUtil的行为。

在使用PowerMock编写测试时，首先需要在测试类上运用框架提供的Annotation：@PrepareForTest，以及一个Runner：PowerMockRunner。因为我们要Mock的对象为EmployeeTableUtil，故而测试类的定义为：
```java
@RunWith(PowerMockRunner.class)
@PrepareForTest(EmployeeTableUtil.class)
public class EmployeeRepositoryTest {
    private EmployeeRepository repository;

    @Before
    public void setUp() throws Exception {
        repository = new EmployRepository();
    }
}
```

现在我要使用PowerMock去Mock静态方法，如EmployeeTableUtil的findAll()方法，至于要测试的方法则为EmployeeRepository的findAll()方法。则编写的单元测试为：
```java
    @Test
    public void should_mock_static_method() {
        List<Employee> employee = new ArrayList<Employee>();
        employee.add(new Employee("1"));
        employee.add(new Employee("2"));

        PowerMockito.mockStatic(EmployeeTableUtil.class);
        when(EmployeeTableUtil.findAll()).thenReturn(employee);

        List<Employee> employees = repository.findAll();
        assertThat(employees.size(), is(2));
        assertThat(employees.get(0).getId(), is("1"));
        assertThat(employees.get(1).getId(), is("2"));

        PowerMockito.verifyStatic();
        EmployeeTableUtil.findAll();
    }
```
Mock静态方法的关键是先要调用框架定义的PowerMockito类的mockStatic()方法（针对EasyMock有相似的类）。方法接收的参数就是我们要Mock的类的类型。接下来就可以调用Mockito框架的方法，对我们要模拟的方法findAll()进行模拟，这里主要的工作是为模拟方法的返回值设置一个stub。之后就是单元测试的验证逻辑。如果需要验证被Mock的方法是否被调用，则需要调用PowerMockito.verifyStatic()方法，紧随其后的是被mock的方法。

如果要Mock的方法是一个命令方法（即没有返回值的方法），做法又有不同。倘若熟悉Mockito，可以看出PowerMock完全沿袭了Mockito的风格（当然，针对EasyMock的扩展则会沿袭EasyMock的风格，这是PowerMock体贴人的地方）：

```java
    @Test
    public void should_mock_exception_for_command_method_in_mock_object() {
        Employee employee = new Employee("1");

        PowerMockito.mockStatic(EmployeeTableUtil.class);
        PowerMockito.doThrow(new NullEmployeeException()).when(EmployeeTableUtil.class);
        EmployeeTableUtil.update(employee);

        assertThat(repository.update(employee), is(false));
    }
PowerMock还可以Mock私有方法，当然只能是实例的私有方法。这主要发生在当我们不希望Mock服务的公开方法时（例如，公开方法的逻辑没有Mock的必要），但这些公开方法的内部又调用了自己的私有方法，而私有方法却需要Mock。例如，EmployeeTableUtil的insert()和delete()方法调用了私有的existed()方法。假设insert()和delete()方法不需要我们Mock，此时就需要对私有方法existed()进行Mock。因为是实例方法，所以下面的测试方法通过调用setTableUtil()方法将被模拟的对象注入到EmployeeRepository对象中：

    @Test
    public void should_mock_private_method() throws Exception {
        Employee employee = new Employee("1");

        EmployeeTableUtil util = PowerMockito.spy(new EmployeeTableUtil());
        PowerMockito.when(util,"existed", anyString())
                .thenReturn(true);

        repository.setTableUtil(util);

        assertThat(repository.insert(employee), is(false));
        assertThat(repository.delete(employee), is(true));
    }
```
PowerMock顺带还提供了测试私有方法的便捷办法（注意是测试，而不是Mock）。例如，测试EmployeeReployee类的私有方法bonus()：

```java
    @Test
    public void should_test_private_method() throws Exception {
        Employee employee = new Employee("1");
        employee.setSalary(8000);

        double result = Whitebox.<Double>invokeMethod(repository, "bonus", employee);
        assertThat(result, is(800d));
    }
```

最后再来看看另外一种诡异的手段。假设我们要测试的方法其内部调用了协作对象的方法，而该协作对象不是在外部注入的，而是在方法中直接实例化。例如在前面例子中，EmployeeRepository的count()方法：

```java
public class EmployeeRepository {
    private EmployeeTableUtil tableUtil;

    public int count() {
        return new EmployeeTableUtil().count();
    }
}
```

要针对这样一种情形进行Mock，做法有所不同。因为它实际针对的是待测类——即这里的EmployeeRepository——执行count()方法，这就需要在count()方法内部形成一个拦截点。因此，需要在@PrepareForTest标记中指向EmployeeRepository类的类型，而非我们要Mock的EmployeeTableUtil。故而，我们需要为这个测试定义一个新的测试类：

```java
@RunWith(PowerMockRunner.class)
@PrepareForTest(EmployeeRepository.class)
public class ConstructionEmployeeRepositoryTest {
    @Test
    public void should_mock_construction_object() throws Exception {
        EmployeeTableUtil util = mock(EmployeeTableUtil.class);
        when(util.count()).thenReturn(100);

        PowerMockito.whenNew(EmployeeTableUtil.class).withNoArguments().thenReturn(util);

        EmployeeRepository repository = new EmployeeRepository();
        assertThat(repository.count(), is(100));
    }
}
```

注意，测试方法的前两行代码调用的mock()与when()方法都是Mockito提供的方法，与PowerMock无关。

我虽然没有看过PowerMock的源代码，但我猜测，当我们在使用PowerMock去Mock静态方法时，定然是结合反射与代理的方式来完成对该方法的调用，其中必然需要初始化该类。由于是静态方法，更多的是需要静态初始化。此外，还有一种情形时，你所要测试的类声明和初始化了一个静态的字段。这些都可能需要调用静态初始化。我们在开发中就碰到一种情形是，我们希望Mock的一个类，定义了一个static块，其中又调用了私有的静态方法。在这个私有静态方法中，依赖了其他的一些对象，这些对象还牵扯到服务容器的问题。即使以静态的方式Mock了该类，仍然逃不过运行static块的命运，换言之，仍然需要依赖服务容器。这时，又可以祭出PowerMock的杀器了。它提供了@SuppressStaticInitializationFor的标注，在该标注中需要传入字符串类型的目标类型的全名。假设EmployeeTableUtil有一个static块是我们需要绕过的，它的类全名为com.agiledon.powermock.EmployeeTableUtil：

```java
@RunWith(PowerMockRunner.class)
@PrepareForTest(EmployeeTableUtil.class)
@SuppressStaticInitializationFor("com.agiledon.powermock.EmployeeTableUtil")
public class EmployeeRepositoryTest {}
```
此外，对于@PrepareForTest以及@SuppressStaticInitializationFor标记而言，如果需要针对多个类型，则需要传入一个数组，例如：

```java
@RunWith(PowerMockRunner.class)
@PrepareForTest({MockedObjectA.class, MockObjectB.class})
@SuppressStaticInitializationFor({"com.agiledon.powermock.MockedObjectA", "com.agiledon.powermock.MockedObjectB"})
public class OneTest {}
```
或许我已经变得像祥林嫂一般的唠叨，但我还是必须再次申明，以上Mock方式所针对的情形皆为设计与代码的坏味道。优先情况下，我们应该重构，使得它遵循DIP原则，解除对服务类的耦合，使其具有良好的可测试性；而不能因为有了强大的PowerMock而“姑息养奸”。换言之，让我们仅仅将PowerMock耍弄的种种花招，看做是压箱底的手段。实在走投无路了，再祭出你的杀手锏吧！
