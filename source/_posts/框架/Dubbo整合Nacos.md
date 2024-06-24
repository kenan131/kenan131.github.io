---
title: Dubbo整合Nacos
date: 2024-02-18 14:17:24
tags: 框架
categories: 框架
---

### 介绍

Apache Dubbo 是一个由阿里巴巴团队最初开发的高性能、轻量级的 Java RPC 框架，用于快速开发高性能的服务。它提供了服务的注册、发现、调用等功能，支持多种语言的实现，但主要是用 Java 编写的。

以下是 Dubbo 的一些主要特性：

1. **服务治理**：Dubbo 提供了服务的注册与发现机制，允许服务消费者和服务提供者在不同的网络环境中进行通信。
2. **负载均衡**：Dubbo 支持多种负载均衡策略，如随机、轮询、最少活跃调用等，以提高系统的可用性和伸缩性。
3. **高度可扩展**：Dubbo 的设计允许开发者通过 SPI 机制轻松扩展其功能，例如自定义的序列化方式、网络传输协议等。
4. **协议支持**：Dubbo 支持多种协议，包括但不限于 Dubbo 协议、HTTP 协议、以及兼容的 Hessian 协议等。
5. **集群容错**：Dubbo 提供了容错机制，如失败重试、快速失败、故障转移等，以提高系统的稳定性。
6. **监控管理**：Dubbo 提供了服务的监控和管理功能，可以通过 Dubbo Admin 等管理界面进行服务的监控和管理。
7. **服务版本**：Dubbo 支持服务版本管理，允许开发者发布和维护不同版本的服务。
8. **服务分组**：Dubbo 允许将服务划分为不同的分组，以适应不同的业务场景。
9. **配置中心集成**：Dubbo 可以与外部配置中心（如 Apollo、Consul、Zookeeper 等）集成，实现动态配置。
10. **安全性**：Dubbo 提供了服务的安全机制，包括权限控制、服务黑白名单等。

Dubbo 适用于需要构建大规模分布式系统的企业级应用，它通过简化分布式服务开发和维护的复杂性，帮助开发者快速构建稳定、高效的服务。

Dubbo 目前已经成为 Apache 软件基金会的一个顶级项目，拥有活跃的社区和持续的更新。

### 使用nacos作为注册中心

新建项目模块，结构如下

```txt
├── dubbo-samples-spring-boot-interface       // 共享 API 模块
├── dubbo-samples-spring-boot-consumer        // 消费端模块
├── dubbo-samples-spring-boot-provider        // 服务端模块
```

在api模块中新建service接口。供服务提供者和服务消费者引用。

```java
public interface DemoService {
    public String sayHi(Integer age,String name);
}
```

#### 提供者

引入依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>


<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-spring-boot-starter</artifactId>
    <version>2.7.23</version>
</dependency>
```

添加配置文件

```yml
dubbo:
  application:
    name: dubbo-springboot-demo-provide
    version: 1.0.0
  scan:
    base-packages: com.example.provide.service
  protocol:
    name: dubbo
    port: -1
  registry:
    address: nacos://localhost:8848

spring:
  application:
    name: dubbo-springboot-demo-provide2

server:
  port: 8081
```

在服务类上添加注解@EnableDubbo。编写实现类代码。

```Java
@SpringBootApplication
@EnableDubbo
public class ProvideApplication {

    public static void main(String[] args) {
        SpringApplication.run(ProvideApplication.class, args);
    }

}

// 实现类上加@DubboService注解
@DubboService
public class DemoProvide implements DemoService {
    @Override
    public String sayHi(Integer age, String name) {
        System.out.println("你好：" + name + "   年龄" + age);
        return "成功！";
    }
}
```

#### 消费者

引入依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>

<!--  dubbo 的依赖 -->
<dependency>
    <groupId>org.apache.dubbo</groupId>
    <artifactId>dubbo-spring-boot-starter</artifactId>
    <version>2.7.23</version>
</dependency>
```

编写配置文件

```yml
dubbo:
  application:
    name: dubbo-springboot-demo-consumer
    version: 1.0.0
  protocol:
    name: dubbo
    port: -1
  registry:
    address: nacos://localhost:8848

spring:
  application:
    name: dubbo-springboot-demo-consumer
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848

server:
  port: 8080
```

编写代码

```java
@SpringBootApplication
@EnableDubbo
public class ConsumerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConsumerApplication.class, args);
    }
}

@Component
public class TestService {
    
    //注入服务
    @DubboReference(loadbalance = "roundrobin")
    private DemoService demoService;

    public void test(Integer arg){
        String result = demoService.sayHi(18, "小王");
        System.out.println(result);
    }
}
```

参考文档：https://cn.dubbo.apache.org/zh-cn/overview/mannual/java-sdk/quick-start/spring-boot/

### Dubbo实现动态指定ip

即消费者可以根据业务场景指定ip进行dubbo调用。借助dubbo插件完成功能。

消费者代码。

```java
public void test(Integer arg){
    ReferenceConfig<DemoService> config = new ReferenceConfig<DemoService>();
    config.setInterface(DemoService.class);
    DemoService demoService1 = config.get();
    UserSpecifiedAddressUtil.setAddress(new Address("192.168.31.1",20881, true));

    String res = demoService1.sayHi(22, "小明");
    System.out.println(res);
}
```

参考文档：https://cn.dubbo.apache.org/zh-cn/overview/mannual/java-sdk/advanced-features-and-usage/service/specify-ip/