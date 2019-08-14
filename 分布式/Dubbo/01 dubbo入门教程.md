

# 简介

Dubbo是一款高性能、轻量级的开源RPC框架，提供了三大核心能力：面向接口的远程方法调用，智能容错和负载均衡，本章旨在帮助大家快速的搭建Dubbo框架服务，文中示例通过SpringBoot+Zookeeper+Dubbo编写代码

# 准备

1. 安装Zookeeper   [zookeeper下载与安装](https://blog.csdn.net/dwl0208/article/details/81807881)

# 搭建环境

## 项目结构

如下图所示，用IDEA创建工程dubbo_hello，并创建三个module

- dubbo_api ：接口包，暴露接口
- dubbo_provider：服务提供者，通过dubbo对外提供服务
- dubbo_consumer : 消费者，通过dubbo使用

![1557158760605](/image/项目结构.png)

## 定义服务接口（dubbo_api)

- pom文件打包方式为jar包

```xml
<groupId>com.xiangyao</groupId>
<artifactId>dubbo_api</artifactId>
<version>0.0.1-SNAPSHOT</version>
<packaging>jar</packaging>
```

- pom文件中添加核心依赖jar包如下：

  - dubbo，引用springboot对应的dubbo依赖

      ```xml
      <dependency>
          <groupId>com.alibaba.spring.boot</groupId>
          <artifactId>dubbo-spring-boot-starter</artifactId>
          <version>2.0.0</version>
      </dependency>
      ```
  
  - zookeeper，引用zookeeper依赖包
  
    ```xml
    <dependency>
        <groupId>org.apache.zookeeper</groupId>
        <artifactId>zookeeper</artifactId>
        <version>3.4.14</version>
    </dependency>
    <dependency>
        <groupId>com.101tec</groupId>
        <artifactId>zkclient</artifactId>
        <version>0.11</version>
    </dependency>
    ```

  

- 定义接口：IDemoService.java

```java
public interface IDemoService {
    String sayHello(String name);
}
```

## 服务提供者(dubbo_provider)

- pom文件中引入`dubbo_api`包

```xml
<dependency>
    <groupId>com.xiangyao</groupId>
    <artifactId>dubbo_api</artifactId>
    <version>0.0.1-SNAPSHOT</version>
</dependency>
```

- 定义配置文件`provider.xml`

  > **注意：** 需引入dubbo配置相关的xml的命名空间【[附录-dubbo命名空间](##dubbo命名空间)】

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://code.alibabatech.com/schema/dubbo
       http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
		
    <!--提供方应用信息 用于计算依赖-->
    <dubbo:application name="hello-world-app"/>
    <!--使用Zookeeper注册中心暴露服务地址-->
    <dubbo:registry address="zookeeper://127.0.0.1:2181"/>
    <!--用dubbo协议在20880端口暴露服务-->
    <dubbo:protocol name="dubbo" port="20880"/>
    <!--声明需要暴露服务的接口-->
    <dubbo:service interface="com.xiangyao.dubbo_api.service.IDemoService" ref="demoService"/>
    <!--和本地bean一样实现服务-->
    <bean id="demoService" class="com.xiangyao.dubbo_provider.service.DemoServiceImpl"/>
</beans>
```

- 定义服务接口实现

  > **注意：** @Service此处注解为`com.alibaba.dubbo.config.annotation.Service`，而非spring的Service注解

```java
@Component
@Service
public class DemoServiceImpl implements IDemoService {
    @Override
    public String sayHello(String name) {
        return "hello " + name;
    }
}
```

- 编写服务启动Application
  1. `@ImportResource`引入依赖的配置文件`provider.xml`

```java
@SpringBootApplication
@ImportResource(value = {"classpath:provider.xml"})
public class DubboProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(DubboProviderApplication.class, args);
        try {
            //阻塞作用，否则会由于不是web项目，执行main方法后立即停止服务。
            System.in.read();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
  2. 启动服务

```java
...
2019-05-07 00:53:51.356  INFO 15664 --- [on(4)-127.0.0.1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2019-05-07 00:53:51.357  INFO 15664 --- [on(4)-127.0.0.1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2019-05-07 00:53:51.373  INFO 15664 --- [on(4)-127.0.0.1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 16 ms
```
## 服务消费者(dubbo_consumer)

- pom文件中引入`dubbo_api`包

```xml
<dependency>
    <groupId>com.xiangyao</groupId>
    <artifactId>dubbo_api</artifactId>
    <version>0.0.1-SNAPSHOT</version>
</dependency>
```

- 定义配置文件`consumer.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://code.alibabatech.com/schema/dubbo
       http://code.alibabatech.com/schema/dubbo/dubbo.xsd">

    <!--消费方应用信息 用于计算依赖 不是匹配条件,不要和提供方一样-->
    <dubbo:application name="consumer-of-helloworld-app"/>
    <!--使用Zookeeper广播注册中心暴露服务地址-->
    <dubbo:registry address="zookeeper://127.0.0.1:2181"/>
    <!--生成远程服务代理，可以和本地bean一样使用demoService-->
    <dubbo:reference id="demoService" interface="com.xiangyao.dubbo_api.service.IDemoService"/>
</beans>
```

- 编写服务启动Application
  1. `@ImportResource`引入依赖的配置文件`consumer.xml`

```java
@SpringBootApplication
@ImportResource(value = {"classpath:consumer.xml"})
public class DubboConsumerApplication {

    public static void main(String[] args) {
        ConfigurableApplicationContext configurableApplication = SpringApplication.run(DubboConsumerApplication.class, args);
        //获取bean实例
        IDemoService demoService = (IDemoService) configurableApplication.getBean("demoService");
        //调用接口
        String result = demoService.sayHello("xianggua");
        System.out.println(result);
    }
}
```
  2. 启动服务，输出结果
```java
...
2019-05-07 01:05:19.999  INFO 16332 --- [tor-EventThread] org.I0Itec.zkclient.ZkClient             : zookeeper state changed (SyncConnected)
2019-05-07 01:05:20.307  INFO 16332 --- [on(2)-127.0.0.1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2019-05-07 01:05:20.307  INFO 16332 --- [on(2)-127.0.0.1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2019-05-07 01:05:20.320  INFO 16332 --- [on(2)-127.0.0.1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 13 ms
hello xianggua
```

# 监控平台

我们可以使用dubbo-admin来监控自己的服务

> **注意：**启动tomcat前，本地需要安装Zookeeper，并启动

- 下载安装，我们从dubbo的源码地址中下载 https://github.com/alibaba/dubbo/releases，下载2.6以下的版本，2.6以上的版本dubbo-admin模块被移除了，我下载了2.5.10版本的

- 解压后，进入dubbo-admin目录，执行命令 **mvn package -Dmaven.test.skip=true**，生成target目录，进入目录，找到dubbo-admin-2.5.10.war文件，把war包拷贝到tomcat目录webapps下

- 启动tomcat，浏览器中输入网址http://localhost:8080/dubbo-admin-2.5.10 ，默认账号密码root/root，看到如下页面表示配置成功

  ![1557163180608](/image/dubbo监控.png)

- 在这个页面可以配置查看自己的服务接口信息，服务提供者，消费者等内容，这里不一一介绍

# 附录：踩坑问题

## dubbo命名空间

- dubbo.xsd命名空间无法通过http://code.alibabatech.com/schema/dubbo/dubbo.xsd进行访问，我们需要本地存储，并通过idea配置依赖关系

![1557161173363](/image/命名空间.png)





