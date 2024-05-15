---
title: spi
date: 2023-05-25 16:04:11
tags: JAVA
categories: JAVA
---

### 介绍

#### 一、什么是SPI机制

SPI是Service Provider Interface 的简称，即服务提供者接口的意思。SPI说白了就是一种扩展机制，我们在相应配置文件中定义好某个接口的实现类，然后再根据这个接口去这个配置文件中加载这个实例类并实例化。有了SPI机制，那么就为一些框架的灵活扩展提供了可能，而不必将框架的一些实现类写死在代码里面。

#### 二、SPI机制的主要目的

- 为了解耦，将接口和具体实现分离开来；
- 提高框架的扩展性。以前写程序的时候，接口和实现都写在一起，调用方在使用的时候依赖接口来进行调用，无权选择使用具体的实现类。

#### 三、SPI机制案例

- JDBC驱动加载案例：利用Java的SPI机制，我们可以根据不同的数据库厂商来引入不同的JDBC驱动包；
- SpringBoot的SPI机制：我们可以在spring.factories中加上我们自定义的自动配置类，事件监听器或初始化器等；
- Dubbo的SPI机制：Dubbo更是把SPI机制应用的淋漓尽致，Dubbo基本上自身的每个功能点都提供了扩展点，比如提供了集群扩展，路由扩展和负载均衡扩展等差不多接近30个扩展点。如果Dubbo的某个内置实现不符合我们的需求，那么我们只要利用其SPI机制将我们的实现替换掉Dubbo的实现即可。

#### 四、如何使用Java的SPI

- 编写功能逻辑代码时，通常我们需要定义一个顶层抽象接口，然后通过实现类来对具体的逻辑代码进行编写。每个实现类都有不同的处理流程。

以支付业务举例

```java
// 支付抽象接口
public interface pay {
    // 支付抽象方法
    void payment();
}
```

编写不同的支付实现类。

```java
// 微信实现类
public class WeiXinPay implements pay{
    @Override
    public void payment() {
        System.out.println("微信支付！");
    }
}
// 支付宝支付类
public class ZhiFuBaoPay implements pay{
    @Override
    public void payment() {
        System.out.println("支付宝支付！");
    }
}
```

编写测试类。

```java
public class TestPay {
    public static void main(String[] args) {
        ServiceLoader<pay> serviceLoader = ServiceLoader.load(pay.class);
        serviceLoader.forEach(pay::payment);
    }
}
```

测试结果

```tex
微信支付！
支付宝支付！
```

