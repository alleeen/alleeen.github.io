# Mock 与 Stub
>在进行单元测试的时候，我们会发现我们要测试的方法会有很多外部依赖，比如：发邮件，进行网络通讯，操作文件系统等等。而我们通常关注的是被测试对象的功能和行为，对于它的依赖，我们仅仅需要关注它们之间的交互，但对依赖的对象是如何执行的具体细节我们并不关注。较为常见的技巧就是使用mock对象或者stub对象来代替真实的依赖。

##Mocks aren't stubs
这是软件大师[Martin Fowler](http://martinfowler.com/)的一篇经典博文。Martin大师在文章中详细的解释了Mock与Stub的区别，以及怎样使用它们进行TDD实践，强烈推荐阅读，猛击[这里](http://martinfowler.com/articles/mocksArentStubs.html)阅读原文。

