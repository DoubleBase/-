## redis cluster 集群架构

支撑N个master node，每个master挂载多个slave node

读写分离的架构，对于每个master来说，写就写到master，然后读就从master对应的slave中去读

高可用，每个master都有slave node，当master挂掉后，redis cluster这套机制会自动将某个slave切换成master

redis cluster：多master+读写分离+高可用



## redis cluster对比replication + sentinal

如果数据量少，主要承载高并发高性能场景，比如缓存就几个G，单击就够了

replication，一个master，多个slave，要几个slave根据实际吞吐量要求来定，然后自己搭建一个sentinal集群，去保证redis主从架构的高可用性，就可以了

redis cluster，主要是针对海量数据+高并发+高可用场景，如果数据量大就建议使用redis cluster



## redis cluster介绍

redis cluster

（1）自动将数据进行分片，每个master上放一部分数据
（2）提供内置的高可用支持，部分master不可用时，还是可以继续工作的

在redis cluster架构下，每个redis要放开两个端口号，比如一个是6379，另外一个就是加10000的端口号，比如16379

16379端口号是用来进行节点间通信的，也就是cluster bus的东西，集群总线。cluster bus的通信，用来进行故障检测，配置更新，故障转移授权

cluster bus用了另外一种二进制的协议，主要用于节点间进行高效的数据交换，占用更少的网络带宽和处理时间



