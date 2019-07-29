## 主从复制的核心原理

- 当启动一个slave node的时候，它会发送一个PSYNC命令给master node

  如果是slave重连master，master仅仅会复制slave部分缺少的数据

  如果是第一次连接，那么会触发master的full resynchronization全量同步，master会启动一个后台线程，生成一个RDB快照文件，同时还会接收客户端的写命令缓存到内存中。RDB生成完毕后，将这个RDB发送给slave，slave会先将RDB持久化到磁盘，再从本地磁盘加载到内存中。然后master会将缓存中的写命令发送给slave，slave也会同步这部分数据

- slave和master之间如果有网络故障，断开了连接，会自动重连，master如果发现有多个slave node都来重连，仅仅会启动一个rdb save操作，用一份数据服务多个slave

![1564409022502](/image/Redis主从复制原理.png)

## 主从复制的断点续传

从redis 2.8开始，就支持主从复制的断点续传，如果主从复制过程中网络连接断了，那么恢复连接后会接着上次复制的地方继续复制，而不是从头开始

master node会在内存中维护一个backlog，master和slave都会保存一个replica offset还有一个master id，replica offset保存在backlog中，如果master和slave连接中断，恢复网络后，slave会让master从上一次的replica offset开始继续复制，如果找不到对应的offset，那么就会执行一次full resynchronization

## 无磁盘化复制

master 在内存中创建rdb，然后发送给slave，不会在自己本地落地磁盘

配置文件参数：

repl-diskless-sync

repl-diskless-sync-delay 5  //等待一定时长再开始复制，因为要等更多slave重新连接过来

## 过期key处理

slave不会过期key，只会等待master过期key。如果master过期了一个key，或者通过LRU淘汰了一个key，那么会模拟一条del命令发送给slave