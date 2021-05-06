# php-7.3.0-ext-curl

curl 扩展长连接， 连接生命周期与php-fpm生命周期一致。

测试：

1. 事先安装libcurl库。

2. 安装php-7.3.0版本

   运行./configure --prefix=/usr/local/php/php7.30 --without-curl –-enable-fpm --enable-debug
   
3. 安装curl扩展

   进入curl目录。
   
   运行/usr/local/php/php7.30/bin/phpize
   
   运行./configure --with-php-config=/usr/local/php/php7.30/bin/php-config --with-curl=/usr/local/lib/curl
   --with-curl是libcurl的安装目录
   
   make & make install
   
4. 抓包测试
   
   i） 编写如下文件curl.php
   
   ![image](https://github.com/11061055/php-7.3.0-ext-curl/blob/master/images/test.png)
   
   
   ii）在php.ini中增加curl扩展，运行如下命令执行文件：
   
   /usr/local/php/php7.30/bin/php -c /usr/local/php/php7.30/etc/php.ini -f curl.php
   
   得到如下结果：
   
   ![image](https://github.com/11061055/php-7.3.0-ext-curl/blob/master/images/result.png)

   可见，close函数并没有释放连接，而是放入连接池缓存中的，两次请求使用的是同一个连接，且连接在5秒后，进程退出的时候释放的。


更多wiki：https://github.com/11061055/php-7.3.0-ext-curl/wiki
