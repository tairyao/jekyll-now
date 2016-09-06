---
layout: post
title: PHP调用Impala SQL—设置执行超时遇到的坑
---

在做哈勃多维监控第二版时，页面的展示数据都由PHP直连Impala执行SQL获取，连接方式用的是官方提供的ODBC扩展。起初并未发现什么问题，但有一次Impala存储集群出问题时，PHP Web机也出现卡死现象。分析发现原因是php-fpm进程过多导致机器资源被耗尽，而php-fpm进程过多是因为Impala集群负载过高，每条sql执行时都被卡死，无法返回结果。所以当用户不断刷新哈勃界面，每个php-fpm进程都一直与Impala保持连接等待其返回结果，php-fpm进程就会越来越多直到pm.max_spare_servers设置的阀值。

