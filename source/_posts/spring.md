---
title: spring
date: 2022-02-16 17:18:02
tags: 框架
categories: 框架
---

### Spring介绍
#### 1.1 概述

Spring：出现在2002年左右，降低企业级开发难度。帮助进行模块之间、类与类之间的管理，帮助开发人员创建对象，管理对象之间的关系。2003年传入国内，被大量使用。2017出现新的流行框架SpringBoot，核心思想与Spring相同。核心技术：IoC、AOP，能使模块之间、类之间解耦合。
依赖：class A使用class B的属性或方法，称之为class A依赖class B。
官网：spring.io

#### 1.2 优点

- 轻量：Spring的所需要的jar包都非常小，一般1M以下，几百kb。核心功能所需要的jar包总共3M左右。
  Spring框架运行占有资源少，运行效率高，不依赖其他jar。
- 针对接口编程，解耦合
- AOP 编程的支持
- 方便集成各种优秀框架

#### 1.3 Spring 体系结构

Spring 由 20 多个模块组成，它们可以分为数据访问/集成（Data Access/Integration）、 Web、面向切面编程（AOP, Aspects）、提供JVM的代理 （Instrumentation）、消息发送（Messaging）、 核心容器（Core Container）和测试（Test）。

### IOC（依赖注入）

- 控制反转（IoC，Inversion of Control），是一个概念，是一种思想。指将传统上由程序代 码直接操控的对象调用权交给容器，通过容器来实现对象的装配和管理。控制反转就是对对 象控制权的转移，从程序代码本身反转到了外部容器。通过容器实现对象的创建，属性赋值， 依赖的管理。

容器概述：

- Context.applicationcontext 接口代表 spring ioc 容器，负责实例化、配置和装配 bean。

- Spring IOC容器就是一个org.springframework.context.ApplicationContext的实例化对象
- 容器负责了实例化，配置以及装配一个bean

ApplicationContext 接口表示 Spring IoC 容器，负责实例化、配置和装配 bean。容器通过读取配置元数据获取关于实例化、配置和组装什么对象的指令。配置元数据用 XML、 Java 注释或 Java 代码表示。它允许您表达组成应用程序的对象以及这些对象之间丰富的相互依赖关系。

### 生命周期

1、postProcessorBeforeInitialization 

2、实例化

3、postProcessorAfterInitialization

3、 初始化（ init-method）

4、afterPropertiesSet 

5、初始化之后@PostConstruct

6、使用

7、销毁 destroy-method

### 三级缓存

- singletonObjects 一级缓存，用于保存实例化、注入、初始化完成的bean实例。
- earlySingletonObjects 二级缓存，用于保存实例化完成的bean实例。
- singletonFactories 三级缓存，用于保存bean创建工厂，以便于后面扩展有机会创建代理对象。

场景：

A 依赖 B & C

B 依赖 A

C 依赖 A

第三级缓存主要是解决的 bean需要创建代理对象的问题。如果不需要创建代理对象则两级缓存即可解决循环依赖问题。

本质则是B去第三级缓存获取工厂后，创建出了代理bean的对象，需要保存在二级缓存中。如果没有保存在二级缓存中，则下一次C去初始化后又会重新获取A工厂并创建bean的代理对象，则会出现多个bean。