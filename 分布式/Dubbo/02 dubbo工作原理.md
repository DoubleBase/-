## dubbo工作原理

dubbo框架设计一共划分了10层，具体如下：

- 第一层：service层，接口层，给服务提供者和消费者来实现的
- 第二层：config层，配置层，主要是对dubbo进行各种配置
- 第三层：proxy层，服务代理层，无论是consumer还是provider，dubbo都会给你生成代理，代理之间进行网络通信
- 第四层：registry层，服务注册层，负责服务的注册与发现你
- 第五层：cluster层，集群层，封装多个服务提供者的路由以及负载均衡，将多个实例组合成一个服务
- 第六层：monitor层，监控层，对rpc接口的调用次数和调用时间进行监控
- 第七层：protocal层，远程调用层，封装rpc调用
- 第八层：exchange层，信息交换层，封装请求响应模式，同步转异步
- 第九层：transport层，网络传输层，抽象mina和netty为统一接口
- 第十层：serialize层，数据序列化层

![dubbo工作原理](/image/dubbo工作原理.png)

工作流程：

- provider向注册中心去注册
- consumer从注册中心订阅服务，注册中心会通知consumer注册好的服务
- consumer调用provider
- consumer和provider都异步通知监控中心

## 注册中心挂了是否可以继续通信

可以，刚开始初始化的时候，消费者会将提供者的地址信息拉取到本地进行缓存，所以注册中心挂了可以继续通信