---
layout: post
title: PHP单元测试之AOP
---

在现代软件工程体系中，单元测试的重要性不言而喻，是我们重构、迁移代码时的安全保障。一般情况下，PHP的单元测试框架都选用PHPUnit。通过PHPUnit提供的桩件（stub），我们可以模拟出一个测试替身来替换被测系统中实际依赖的组件。而在桩件的基础上，仿件（Mock）模拟出来的测试替身还能验证其预期的行为，比如断言某个方法必定会被调用一次。

但是PHP没有像JAVA等语言的强设计模式等特性，一般的PHP WEB框架也没有做强制要求，导致我们在实现代码功能时会随手new一个类来使用，而没有做依赖注入。对于这类场景的方法做单元测试，PHPUnit就无能为力了。

所以这里隆重推荐AspectMock，它在PHPUnit的基础上，集成了PHP的Go-AOP框架，因此具备切面注入替换原有对象的功能，让没有显示依赖注入的方法也能做单元测试。

AspectMock支持的特性如下：

- Create test doubles for static methods.

- Create test doubles for class methods called anywhere.

- Redefine methods on the fly.

- Simple syntax that's easy to remember.

我们拿ThinkPHP框架来举个例子（Laravel等现代框架本身自带完善的单元测试功能，不在此讨论范围内）。