# Zookeeper

## 安装目录

`/usr/local/src/zookeeper-3.4.9`

## 服务端命令

- 启动ZK服务：`sh zkServer.sh start`
- 查看ZK服务状态：`sh zkServer.sh status`
- 停止ZK服务：`sh zkServer.sh stop`
- 重启ZK服务：`sh zkServer.sh restart`
- 查看是否有ZK服务：`ps -ef | grep zookeeper` 或 `jps`

## 客户端命令

- 连接zk：`sh zkCli.sh -server ip:port`

- 查看目录包含的所有文件：`ls`

- 查看某个目录包含的所有文件：`ls2 /`

- 创建znode，并设置初始化内容：`create [-s] [-e] /test "hello"`

  -s -e分别指定节点特性：顺序或临时节点，不指定则默认创建的是永久节点

- 获取znode数据：`get /test`

- 设置znode内容：`set /test "update hello"`

- 删除znode节点：`delete /test [version]`

  加了version表示删除指定版本的节点

- 关闭连接：`close`

- 连接ZK：`connect host:port`

- 退出客户端：`quit`



# 集群模式

- 至少准备3台服务器，每天服务器上配置zoo.cfg文件，按照如下配置数据和日志目录，以及服务器的IP

  > dataDir=E:\\software\\zkcluster\\3001\\data
  > dataLogDir=E:\\software\\zkcluster\\3001\\log
  >
  > server.1=127.0.0.1:2888:3888
  > server.2=127.0.0.1:2889:3889
  > server.3=127.0.0.1:2890:3890

  其中 server.id=ip:port:port 表示一台服务器，id被称为serverID，标识服务器在集群中的机器序号，同时要在数据目录dataDir下创建一个myid文件，里面只有一行内容，并且是一个数字，即对应每一台机器的serverId数字

- 每台服务器上的zoo.cfg文件内容都应该一致，因此最好使用SVN或是GIT把此文件管理起来，确保每个机器都能共享到一份相同的配置。

- myid文件中只有一个数字，需要确保每台服务器的myid文件的数字不同，并保证和自己服务器上zoo.cfg文件中的serverid一致，id范围为1~255




# API

- 客户端可以对一个不存在的节点进行子节点变更的监听
- 一旦客户端对一个节点注册了子节点列表变更监听之后，那么当该节点的子节点列表发生变更的时候，服务端都会通知客户端，并将最新的子节点列表发送给客户端
- 该节点本身的创建和删除也会通知客户端



Curator提供了一套易用性和可读性更强的API，提供了Zookeeper各种应用场景（Recipe，如共享锁服务、Master选举机制和分布式计数器等）的抽象封装



- Master选举
- 分布式锁
- 分布式计数器
- 分布式Barrier



TestingServer：简易的Zookeeper服务，用来单元测试

TestingCluster：模拟Zookeeper集群环境