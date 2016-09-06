---
layout: post
title: PHP调用Impala SQL—设置执行超时遇到的坑
---

| 导语 虽然PHP语言本身有诸多限制，但遇到问题时如果能坚持一步一步深挖下去，总能把问题解决，折腾过程中也能学到很多跨语言的通用技能。

在做哈勃多维监控第二版时，页面的展示数据都由PHP直连Impala执行SQL获取，连接方式用的是官方提供的ODBC扩展。起初并未发现什么问题，但有一次Impala存储集群出问题时，PHP Web机也出现卡死现象。分析发现原因是php-fpm进程过多导致机器资源被耗尽，而php-fpm进程过多是因为Impala集群负载过高，每条sql执行时都被卡死，无法返回结果。所以当用户不断刷新哈勃界面，每个php-fpm进程都一直与Impala保持连接等待其返回结果，php-fpm进程就会越来越多直到pm.max_spare_servers设置的阀值。

如何优化？

## 方案一：设置单个PHP请求超时时间 ##

先用最简单、最快速的方式解决问题，让php-fpm进程超时后主动结束。设置PHP执行超时有如下几个参数：

- php.ini中的max_execution_time参数：统计的是php进程真正占用cpu的时间，不符合当前场景，无效；

- php的set_time_limit()函数：设置当前php脚本执行的最长时间，但是只统计php本身执行的时间，不包括使用system()等系统调用、流操作、数据库操作等。所以也无效；

- php-fpm.conf的request_terminate_timeout参数：设置一个php-fpm请求处理的最长时间，超时后返回404。符合当前需求，设为90秒。

这样就算初步解决了问题，当php-fpm进程越来越多，且都在等待Impala返回时，到了90秒的超时时间，php-fpm进程就开始主动断开与Impala的连接并返回404，这时候新的用户请求又可以进来了，如果Impala恢复，那么Web服务也会自动恢复。

## 方案二：设置Impala SQL执行超时 ##

方案一中存在明显缺陷，就是HTTP返回码是404，php本身处理也被中断，没有机会把处理结果上报到Monitor做监控和后续分析。那么是否可以在php调用Impala时设置sql的执行时间？先看php执行Impala sql的整个调用链：phpodbc->unixodbc->impalaodbc。涉及到如下几个参数：

- odbc.ini中SocketTimeout参数：The number of seconds after which Impala closes the connection with the client application if the connection is idle。统计连接空闲时间，无效；

- odbc_setoption()方法：测试无效，估计是impalaodbc没有实现unixodbc这一接口；

- unixodbc的DMStmtAttr=SQL_QUERY_TIMEOUT=10参数：同样测试无效，impalaodbc的说明文档中都未提及这一参数；

- 执行sql前先设置SET QUERY_TIMEOUT_S=10参数：The timeout clock for queries and sessions only starts ticking when the query or session is idle。也是统计连接空闲时间，无效。

方案二完全失败，无论是从odbc调用客户端，还是Impala服务端，都无法设置sql超时时间。这一点Impala的odbc实现的不如mysqli等扩展好。

## 方案三：设置PHP函数执行超时 ##

既然不能设置Impala sql的执行超时，那就从php的函数入手，如果能设置php的函数执行超时，那效果也是一样的。设置php函数超时的方式主要如下几种：

- pcntl_alarm()设置闹钟超时信号，pcntl_signal()捕获闹钟信号后抛出异常，达到终止函数运行的效果。
declare(ticks = 1);
pcntl_signal(SIGALRM, function() {throw new Exception('process timeout');});

		try {
		    pcntl_alarm(1);
		    test();
		} catch (Exception $e) {
		    print $e->getMessage()."\n";
		}

		function test() {
		    sleep(3);
		}

- libevent定时器。既然pcntl_alarm()定时被重置，那试试libevent定时器能否满足需求。

		$base = event_base_new();
		$event = event_new();
		event_set($event, 0, EV_TIMEOUT, function() {
		    print "time out\n";
		});
		event_base_set($event, $base);
		event_add($event, 2000000);        //2秒超时
		//event_base_loop($base);        //默认阻塞，会在这里等待2秒
		event_base_loop($base, EVLOOP_NONBLOCK);
		print "continue";
		sleep(3);
		event_base_loop($base);
		print "end";

- swoole异步定时器。看看重新定义PHP的swoole大法行不行。

		swoole_timer_after(1, function(){
		    print "time out\n";
		});
		sleep(3);
		print "end\n";    //因为sleep()不是异步执行的，所以先输出end，最后才输出time out

