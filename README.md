# php-7.3.0-ext-curl

curl 保持长连接， 连接生命周期与php-fpm生命周期一致。

想法产生于一次服务性能profile。

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
