---
layout: post
title: PHP调用Impala SQL—设置执行超时遇到的坑
---

导语 — 虽然PHP语言本身有诸多限制，但遇到问题时如果能坚持一步一步深挖下去，总能把问题解决，折腾过程中也能学到很多跨语言的通用技能。

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

以上代码执行无误，当程序执行1秒后输出process timeout并退出。但是当把操作Impala的代码放到test()中后，发现超时效果没有了。查看pcntl_alarm()函数的实现，原来同一个进程每次对pcntl_alarm()的调用都会取消之前设置的alarm信号，包括php中调用的C语言扩展中使用的alarm()函数。所以在phpodbc->unixodbc->impalaodbc这条调用链中，必然有一步是调用了alarm()函数。查看了phpodbc和unixodbc的C源码，并未发现，而impalaodbc没有提供源码，查看不了。失败；

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

event_base_loop默认是阻塞执行的，不满足需求。当设置EVLOOP_NONBLOCK参数时，变为非阻塞直接往下运行，但仅仅只是检测是否有事件就绪，如果有则触发，没有就再也没有然后了，不会像pcntl_alarm()那样超时后可以打断程序的执行，所以也无法满足需求。失败；

- swoole异步定时器。看看重新定义PHP的swoole大法行不行。

		swoole_timer_after(1, function(){
		    print "time out\n";
		});
		sleep(3);
		print "end\n";    //因为sleep()不是异步执行的，所以先输出end，最后才输出time out

swoole的定时器要求后续的代码也都是异步的，否则定时器要最后才被触发。当前swoole官方只实现了mysql、redis等少数几个异步扩展，所以这条路也走不通。失败。

方案三完全失败，当前场景无法给php函数设置执行超时时间。

## 方案四：PHP多进程实现Impala执行超时 ##

既然一个进程搞不定，那用两个进程总可以了吧。PHP的多线程支持不好，但pcntl多进程实现还算可以。通过pcntl_fork()产生一个子进程，由子进程执行Impala sql，父进程监控执行时间，如果超时就调用posix_kill()干掉子进程。如果子进程超时前能执行完，则把结果传递给父进程。父子进程通信方式有以下几种：

- 子进程把odbc_exec()执行返回的资源句柄传递给父进程，父进程再由自己odbc_fetch_array()获取数据，这样传递的数据量最小。但是php不支持对资源类型进行序列化，且反序列化时还需要类的定义代码，所以根本实现不了，只能通过子进程执行完获取到全部数据后再传递给父进程；

- 管道通信。

		$time = time();
		$pid = pcntl_fork();
		if ($pid == -1) {
		        print "Fork failed!\n";
		} elseif ($pid == 0) {
		        //child
		        $childPid = posix_getpid();
		        $pipePath = "/tmp/{$childPid}{$time}.pipe";
		        createPipe($pipePath);

		        $result = odbc_exec($conn, $sql);
		        $data = array();
		        while($row = odbc_fetch_array($result)) {
		                $data[] = $row;
		        }
		        $data = json_encode($data);
		        $size = mb_strlen($data, 'UTF-8');    //计算长度
		        $splitSize = 65536;    //每次只传输65536 bytes，但超过65536好像也没问题
		        $start = 0;
		        $file = fopen($pipePath, 'w');
		        while ($start < $size) {
		                $splitData = mb_substr($data, $start, $splitSize);
		                fwrite($file, $splitData);
		                $start += $splitSize;
		        }
		        exit(0);
		} else {
		        //parent
		        try {
		                pcntl_alarm(15);
		                //pcntl_wait($status);        //管道自带阻塞属性，所以这里不用wait子进程结束
		                $pipePath = "/tmp/{$pid}{$time}.pipe";
		                while (!file_exists($pipePath)) {
		                        usleep(1000);        //坐等子进程创建好管道文件
		                }
		                $file = fopen($pipePath, 'r');
		                $data = '';
		                while ($json = fread($file, 65535)) {    //每次读取65536 bytes
		                        $data .= $json;
		                }
		                print_r(json_decode($data, true));
		                unlink($pipePath);
		        } catch (Exception $e) {
		                print $e->getMessage()."\n";
		                posix_kill($pid, 9);
		                print "kill $pid\n";
		        }
		}

- 共享内存通信。

		$time = time();
		$pid = pcntl_fork();
		if ($pid == -1) {
		        print "Fork failed!\n";
		} elseif ($pid == 0) {
		        //child
		        $childPid = posix_getpid();
		        $result = odbc_exec($conn, $sql);
		        $data = array();
		        while($row = odbc_fetch_array($result)) {
		                $data[] = $row;
		        }
		        $data = json_encode($data);
		        $size = mb_strlen($data, 'UTF-8');
		        $shmid = shmop_open($childPid.$time, 'c', 0644, $size);
		        shmop_write($shmid, $data, 0);
		        shmop_close($shmid);
		        exit(0);
		} else {
		        //parent
		        try {
		                pcntl_alarm(15);        //设置超时
		                pcntl_wait($status);
		                $shmid = shmop_open($pid.$time, 'w', 0, 0);
		                $size = shmop_size($shmid);
		                $data = shmop_read($shmid, 0, $size);
		                $data = json_decode($data, true);
		                print_r($data);
		                shmop_delete($shmid);
		                shmop_close($shmid);
		        } catch (Exception $e) {
		                print $e->getMessage()."\n";
		                posix_kill($pid, 9);
		                print "kill $pid\n";
		        }
		}

CLI下测试ok，共享内存也是效率最高的方式。

命令行下测试时信心满满，准备嵌入到Web框架代码中，才想起来php-fpm下不支持再用多进程的方式处理请求。不过既然都写好代码了，就迁移上去试一下吧，果然出现了各种奇葩问题。

- 子进程结束后，父进程mysql连接丢失。这一点在预期内；

- 父进程执行时间会跟pcntl_alarm()设置有关，而不是正常结束；

- 子进程一旦结束，nginx就直接返回结果了，而父进程还在运行。

既然官方文档都说了不要使用这种方式，那再折腾下去也没有意义了。方案四失败。

## 方案五：PHP通过Thrift执行Impala SQL ##

之前的方案都是通过ODBC在跟Impala打交道，那能不能换种方式呢？Impala也是属于Hadoop平台体系的，所以也支持Thrift协议。Github搜一下，果然有个php通过Thrift操作Impala的项目，clone下来试了一下，Thrift和Impala版本太老，运行不起来。去cloudera公司的Github下找到了python版本的实现，还有第三方的java实现，所以就能参照这些自己重新实现了。

		require_once '/home/tairyao/thrift-0.9.3/lib/php/lib/Thrift/ClassLoader/ThriftClassLoader.php';
		use Thrift\ClassLoader\ThriftClassLoader;
		use Thrift\Protocol\TBinaryProtocol;
		use Thrift\Transport\TSocket;
		use Thrift\Transport\TBufferedTransport;
		use Thrift\Exception\TException;

		$GEN_DIR = realpath(dirname(__FILE__)).'/gen-php';
		$loader = new ThriftClassLoader();
		$loader->registerNamespace('Thrift', '/home/tairyao/thrift-0.9.3/lib/php/lib');
		$loader->registerDefinition('impala', $GEN_DIR);
		$loader->registerDefinition('beeswax', $GEN_DIR);
		$loader->register();

		try {
		    $socket = new TSocket( 'xxx', '21000' );
		    $socket->setRecvTimeout(15000);

		    $transport = new TBufferedTransport($socket, 1024, 1024);
		    $protocol = new TBinaryProtocol($transport);
		    $client = new \impala\ImpalaServiceClient($protocol);
		    $transport->open();

		    $query = new \beeswax\Query(array('query' => 'use mmdb'));
		    $client->query($query);

		    $sql = "select * from t_mm_dispatch_wns limit 10";
		    $query = new \beeswax\Query(array('query' => $sql));
		    $handle = $client->query($query);

		    $metadata = $client->get_results_metadata($handle);
		    $fields = $metadata->schema->fieldSchemas;

		    $data = array();
		    $dataPiece = $client->fetch($handle, false, 1024);
		    $data = array_merge($data, parseField($dataPiece, $fields));
		    while ($dataPiece->has_more){
		        $dataPiece = $client->fetch($handle, false, 1024);
		        $data = array_merge($data, parseField($dataPiece, $fields));
		    }
		    $transport->close();
		    print_r($data);

		} catch (\Exception $e) {
		    print $e->getMessage() . "\n";
		}

		function parseField($data, $fields) {
		    $result = array();
		    $fieldCount = count($fields);
		    foreach ($data->data as $row => $rawValues) {
		        $values = explode("\t", $rawValues, $fieldCount);
		        foreach ($fields as $k => $fieldObj) {
		            $result[$row][$fieldObj->name] = $values[$k];
		        }
		    }
		    return $result;
		}

需要注意的地方：

- Thrift文件参照python版本，需要修改里面的namespace为php格式；

- Impala端口。

	21050：for JDBC or ODBC version 2.x driver；

	21000：for beeswax (impala-shell、thrift) or original ODBC driver；

- 自动生成的thrift php接口代码中，有可能把php关键字定义成方法名了，需要修改下。比如echo，GLOBAL；

- 字段名和数据是两个接口返回的，需要自己组装。几乎不影响性能；

- 数据是一行记录，需要自己通过\t分割，如果字段内容中有\t，就会出问题；

- 一次只能fetch 1024行记录，如果没fetch完，cdh管理平台上会显示执行错误。

而Thrift设置超时就方便多了，这里只需$socket->setRecvTimeout()即可。折腾了这么多，方案五总算成功满足了业务的需求。

最后，不要觉得平时只写业务逻辑代码没有成长，如果能抓住一些小问题深挖下去，还是能学到不少知识的。特别是对于一些可做可不做的优化点，如果每次都逃避过去当然不会有成长，而是要主动挑战它。另外，平时的难点解决过程也很值得记录下来，因为不管是晋升还是跳槽面试，面试官最喜欢问的问题之一就是你遇到过什么难题、是怎么解决的？
