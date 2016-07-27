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

1. PHPUnit的安装省略；
2. AspectMock的安装参考：github链接；
3. 建立一个Test目录，放入以下文件：
	- phpunit.xml：phpunit全局设置
	```
    <?xml version="1.0" encoding="UTF-8" ?>
    <phpunit bootstrap="aspectMockAutoload.php" backupGlobals="false">
    </phpunit>
	```
	这里指定phpunit加载的bootstrap文件为aspectMockAutoload.php
	- aspectMockAutoload.php：AspectMock的引导程序
	```
	<?php
	preg_match("/(.*)\\/Application\\/.*/", __DIR__, $match);
	$tpPath = $match[1];
	include $tpPath.'/vendor/autoload.php';     // composer autoload

	$kernel = \AspectMock\Kernel::getInstance();
	$kernel->init([
	    'debug' => true,
	//    'includePaths' => [__DIR__],
	]);

	//这里要用绝对路径，否则在不同目录下执行phpunit，获取的相对路径都不同
	$kernel->loadFile(__DIR__.'/bootstrap.php');
	```
	- bootstrap.php：ThinkPHP框架的引导程序（略）
	- ApiControllerTest.php：单元测试例子。同一个项目的单元测试文件都放在当前路径下
	```
	<?php
	use AspectMock\Test as test;
	class ApiControllerTest extends PHPUnit_Framework_TestCase {
	    protected $controller;
	    protected function setUp() {
	        $this->controller = new \Home\Controller\ApiController();
	    }
	    protected function tearDown() {
	        test::clean(); // remove all registered test doubles
	    }
	    public function testGetAll() {
	        //mock一个Model类的替身，设置query方法返回abc
	        $model = test::double('\Think\Model', ['query' => 'abc']);
	        //无需把替身注入到controller中，AOP机制会在test()方法中自动把Model类替换为替身
	        $data = $this->controller->getAll();
	        print_r($data);
	        //断言结果为abc
	        $this->assertEquals('abc', $data);
	        //断言query方法被反射调用了一次
	        $model->verifyInvokedOnce('query');
	    }
	}
	```

4. 测试单个文件：

```
[root@localhost Test]# pwd
/home/tairyao/ThinkPHPUnitTest/Application/Home/Test
[root@localhost Test]# phpunit ApiControllerTest.php
PHPUnit 4.8.9 by Sebastian Bergmann and contributors.

.abc

Time: 381 ms, Memory: 56.25Mb

OK (1 test, 1 assertion)
```

5. 测试整个目录：

```
[root@localhost Home]# pwd
/home/tairyao/ThinkPHPUnitTest/Application/Home
[root@localhost Home]# phpunit -c Test/phpunit.xml Test/
PHPUnit 4.8.9 by Sebastian Bergmann and contributors.

.abc

Time: 458 ms, Memory: 56.50Mb

OK (1 test, 1 assertion)
```

虽然扩展框架可以让我们有办法对各种不规范的代码进行单元测试，但我们更应该在源头上遏制这种不规范的产生，写出TDD（测试驱动开发）的代码。最后附上几条个人总结的关于写出可测试PHP代码的建议：

- 每个单元的环境要隔离，单元之间尽量减少耦合（单一职责原则），有明确依赖关系（依赖注入）；
- 每个方法的职责单一，尽量保持简单，太大就拆分。功能越多，依赖也就越多，代码维护混乱；
- 不要使用全局变量、静态方法等，会造成隐性依赖。有些依赖框架的常量、函数等需要先加载框架才能使用，对测试也不友好；
- Singleton单例模式的getInstance()也是全局的；
- 不要在方法中直接new，依赖注入到__construct()或方法的形参中；
- __construct()中不要有业务逻辑；
- 不要使用exit粗暴地打断代码处理流，用return或自定义异常代替；
- 不要使用php的魔术方法，增加对运行时数据的耦合。