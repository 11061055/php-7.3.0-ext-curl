# php-7.3.0-ext-curl

curl 保持长连接， 连接生命周期与php-fpm生命周期一致。

因为工作中不仅涉及业务开发，也涉及服务稳定性和服务规划，所以经常面临一些问题，这个想法产生于一次性能profile。

在用xprof对服务进行性能profile时，发现这三项非常耗时：

1. redis查询
2. mysql查询
3. 外部服务调用

现状：

1. redis用于做缓存和一些业务数据，但是redis和业务服务器不在同一个网段，一次redis查询RTT为5ms，而一个请求可涉及50个查询。
2. mysql与业务服务器也不在同一个网段。
3. 依赖的外部服务与服务器不在同一个地区，且走的是公网https通道，一个RTT为30ms。

优化：

1. redis和业务服务器同机房部署。
2. mysql查询结果做redis缓存和代码级缓存。
3. 公网https通道的额外消耗在每次短连接的tcp握手和ssl握手，所以将短连接改为长连接后可以节约tcp握手和ssl握手。


测试：

1. 事先安装libcurl库。

2. 安装php-7.3.0版本

   压缩文件见上面代码仓库，解压。
   运行./configure --prefix=/usr/local/php/php7.30 --without-curl –-enable-fpm --enable-debug
   不安装curl扩展，安装php-fpm，开启debug模式(开发的时候，如果使用源码提供的内存管理工具，会报告是否内存泄露).
   
3. 安装curl扩展

   用本仓库的interface.c php_curl.h 替换原有的curl扩展对应文件，进入curl目录。
   运行/usr/local/php/php7.30/bin/phpize
   运行./configure --with-php-config=/usr/local/php/php7.30/bin/php-config --with-curl=/usr/local/lib/curl
   --with-curl是libcurl的安装目录
   make & make install
   
4. 抓包测试
   
   i） 编写如下文件curl.php
   
   <?php

   echo "\n-------------------------------------------------\n\n";

   $curlobj = curl_init_p("http://39.156.69.79/");
   $rtn = curl_exec($curlobj);
   curl_close($curlobj);

   echo "\n-------------------------------------------------\n\n";

   $curlobj = curl_init_p("http://39.156.69.79/");
   $rtn = curl_exec($curlobj);
   curl_close($curlobj);


   echo "\n-------------------------------------------------\n\n";
   
   ?>
   
   
   ii）在php.ini中增加curl扩展，运行如下命令执行文件：
   
   /usr/local/php/php7.30/bin/php -c /usr/local/php/php7.30/etc/php.ini -f curl.php
   
   得到如下结果：：

-------------------------------------------------

11:32:26.198309 IP 192.168.0.102.51205 > 39.156.69.79.http: Flags [S], seq 16760035, win 65535, options [mss 1460,nop,wscale 6,nop,nop,TS val 277858728 ecr 0,sackOK,eol], length 0
11:32:26.207817 IP 39.156.69.79.http > 192.168.0.102.51205: Flags [S.], seq 733639993, ack 16760036, win 8192, options [mss 1412,nop,wscale 5,nop,nop,nop,nop,nop,nop,nop,nop,nop,nop,nop,nop,sackOK,eol], length 0
11:32:26.207886 IP 192.168.0.102.51205 > 39.156.69.79.http: Flags [.], ack 1, win 4096, length 0
11:32:26.208112 IP 192.168.0.102.51205 > 39.156.69.79.http: Flags [P.], seq 1:52, ack 1, win 4096, length 51: HTTP: GET / HTTP/1.1
11:32:26.220965 IP 39.156.69.79.http > 192.168.0.102.51205: Flags [.], ack 52, win 772, length 0
11:32:26.220973 IP 39.156.69.79.http > 192.168.0.102.51205: Flags [P.], seq 1:306, ack 52, win 772, length 305: HTTP: HTTP/1.1 200 OK
11:32:26.220975 IP 39.156.69.79.http > 192.168.0.102.51205: Flags [P.], seq 306:387, ack 52, win 772, length 81: HTTP
11:32:26.221058 IP 192.168.0.102.51205 > 39.156.69.79.http: Flags [.], ack 306, win 4091, length 0
11:32:26.221090 IP 192.168.0.102.51205 > 39.156.69.79.http: Flags [.], ack 387, win 4089, length 0
<html>
<meta http-equiv="refresh" content="0;url=http://www.baidu.com/">
</html>

-------------------------------------------------

11:32:26.221748 IP 192.168.0.102.51205 > 39.156.69.79.http: Flags [P.], seq 52:103, ack 387, win 4096, length 51: HTTP: GET / HTTP/1.1
11:32:26.230722 IP 39.156.69.79.http > 192.168.0.102.51205: Flags [.], ack 103, win 772, length 0
11:32:26.230734 IP 39.156.69.79.http > 192.168.0.102.51205: Flags [P.], seq 387:692, ack 103, win 772, length 305: HTTP: HTTP/1.1 200 OK
11:32:26.230823 IP 192.168.0.102.51205 > 39.156.69.79.http: Flags [.], ack 692, win 4091, length 0
<html>
<meta http-equiv="refresh" content="0;url=http://www.baidu.com/">
</html>
11:32:26.231487 IP 39.156.69.79.http > 192.168.0.102.51205: Flags [P.], seq 692:773, ack 103, win 772, length 81: HTTP
11:32:26.231600 IP 192.168.0.102.51205 > 39.156.69.79.http: Flags [.], ack 773, win 4094, length 0

-------------------------------------------------

11:32:31.237241 IP 192.168.0.102.51205 > 39.156.69.79.http: Flags [F.], seq 103, ack 773, win 4096, length 0
11:32:31.246904 IP 39.156.69.79.http > 192.168.0.102.51205: Flags [.], ack 104, win 772, length 0
11:32:31.246910 IP 39.156.69.79.http > 192.168.0.102.51205: Flags [F.], seq 773, ack 104, win 772, length 0
11:32:31.246971 IP 192.168.0.102.51205 > 39.156.69.79.http: Flags [.], ack 774, win 4096, length 0

   可见，两次请求使用的是同一个连接，且连接在5秒后，进程退出的时候释放的。
