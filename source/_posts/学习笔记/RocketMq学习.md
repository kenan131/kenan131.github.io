---
title: RocketMq
date: 2023-4-06 16:09:04
tags: 学习笔记
categories: 学习笔记
---

### 介绍

Apache RocketMQ是一个开源的分布式消息队列系统，由中国的阿里巴巴集团开发并贡献给Apache软件基金会。它最初是为了满足阿里巴巴内部大规模的异步消息传输需求而设计的，现在已经成为一个广泛使用的中间件产品，适用于多种场景，包括异步通信、应用解耦、消息流处理、事件驱动架构等。

以下是RocketMQ的一些关键特性：

1. **高性能**：RocketMQ提供了高性能的消息处理能力，能够支持每秒数百万级别的消息吞吐量。
2. **高可用性**：通过主从复制、集群部署和故障自动转移等机制，RocketMQ确保了服务的高可用性。
3. **消息持久化**：RocketMQ支持将消息持久化到磁盘，确保消息不会因为系统故障而丢失。
4. **灵活的消息模型**：RocketMQ支持多种消息类型，包括普通消息、顺序消息、延迟消息和事务消息等。
5. **大规模集群支持**：RocketMQ设计用于大规模分布式系统，可以水平扩展以支持更多的消息处理。
6. **多种消费模式**：支持集群消费模式和广播消费模式，以适应不同的业务需求。
7. **消息过滤**：消费者可以根据消息的标签或属性进行过滤，只消费感兴趣的消息。
8. **顺序消息**：RocketMQ提供了严格的顺序消息功能，确保在同一个队列中消息的顺序性。
9. **事务消息**：支持分布式事务，确保消息的发送和业务操作的原子性。
10. **灵活的部署方式**：支持多种部署方式，包括单节点、多节点、伪集群和云原生部署等。
11. **丰富的客户端支持**：提供了Java、Python、Go等多种语言的客户端SDK。
12. **监控和运维**：RocketMQ提供了丰富的监控指标和运维工具，方便系统的监控和管理。

RocketMQ适用于需要高吞吐量、高可靠性和灵活消息处理能力的场景，例如电子商务、金融、物联网等行业的大规模消息系统。随着技术的发展和社区的贡献，RocketMQ不断增加新的特性和优化，以满足更广泛的应用需求。

#### RocketMQ的架构及消息存储

RocketMQ的部署架构如下图所示，在早期的RocketMQ版本中，是有依赖ZK的。而现在的版本中已经去掉了对ZK的依赖，转而使用自己开发的NameServer来实现元数据（Topic路由信息）的管理，并且这个NameServer是无状态的，可以随意的部署多台，其代码也非常简单，非常轻量。

NameServer内部维护了topic和broker之间的对应关系，并且和所有broker保持心跳连接，producer和consumer需要发布或者消费消息的时候，向NameServer发出请求来获取连接的broker的信息。

RocketMQ的启动流程可以描述为：

- Broker消息服务器启动，向所有NameServer注册，NameServer与每台Broker服务器保持长连接，并定时检测 Broker是否存活，如果检测到broker宕机，则从路由注册表中将其移除。
- 消息生产者(Producer)在发送消息之前先从NameServer获取Broker服务器地址列表，然后根据负载算法从列表中选择一台消息服务器进行消息发送。
- 消息消费者（Consumer）在拉取消息之前先从NameServer获取Broker服务器地址列表，然后根据负载算法从列表中选择一台消息服务器进行消息拉取。

#### Topic 

Apache RocketMQ 中消息传输和存储的顶层容器，用于标识同一类业务逻辑的消息。主题通过TopicName来做唯一标识和区分。

#### CommitLog

消息主体以及元数据的存储主体。大白话，就是存储生产者生成的消息内容。

在RocketMQ中所有的topic的消息数据都会存储在一个CommitLog文件中。在消息写入磁盘的时候采用顺序写。

#### messageQueue

一个Topic对用多个MessageQueue，也就是topic会存在不同的broker中。

####  ConsumeQueue 是什么
ConsumeQueue，又称作消费队列，是 RocketMQ 存储系统的一部分，保存在磁盘中。该文件可以看作 CommitLog 关于消息消费的“索引”文件。ConsumeQueue 是一个 MappedFileQueue，即每个文件大小相同的内存映射文件队列。每个文件由大小和格式相同的索引项构成。每一个 Topic 的 Queue，都对应一个 ConsumeQueue。

**ConsumeQueue 的作用**

- 引入 ConsumeQueue 的目的主要是适应消息的检索需求，提高消息消费的性能。Broker 中所有 Topic 的消息都保存在 CommitLog 中，所以同一 Topic 的消息在 CommitLog 中不是连续存储的。消费某一 Topic 消息时去遍历 CommitLog 是非常低效的，所以引入了 ConsumeQueue。
- 一个 ConsumeQueue 保存了一个 Topic 的某个 Queue 下所有消息在 CommitLog 中的起始物理偏移量offset，消息大小size和消息Tag的HashCode值。当需要消费这个 Topic 时，只需要找到对应的 ConsumeQueue 开始遍历，根据消息在 CommitLog 中的偏移量即可找到消息保存的位置。

### 消息处理过程

在Apache RocketMQ中，Broker不会主动推送消息给消费者。相反，消费者通过拉取（pull）的方式从Broker获取消息。以下是消息传递过程的详细说明：

1. **生产者发送消息**：生产者将消息发送到Broker。Broker将消息存储在对应的Topic的队列中。
2. **消费者拉取消息**：消费者通过拉取请求（Pull Request）向Broker请求消息。消费者可以指定从哪个队列、哪个偏移量（offset）开始拉取消息。
3. **长轮询机制**：如果Broker上没有新消息可供拉取，消费者会等待一段时间（长轮询等待时间），然后再次尝试拉取。这样可以减少空轮询的次数，提高效率。
4. **消息确认**：消费者在处理完消息后，需要向Broker发送消息确认（acknowledgment）。Broker收到确认后，才会认为消息已经被成功消费，并从队列中移除。
5. **消费模式**：RocketMQ支持两种消费模式：
   - **集群消费模式**：消费者分布在不同的Broker上，每个消费者只消费部分消息。
   - **广播消费模式**：所有消费者都会收到所有消息，适用于需要多个消费者处理相同消息的场景。
6. **消息顺序**：在同一个队列中，消息是有序的。如果消费者订阅了同一个Topic的多个队列，Broker会保证每个队列内的消息顺序，但不同队列之间的消息顺序可能不一致。
7. **消息过滤**：消费者可以通过设置消息标签（tag）或表达式来过滤消息，只拉取满足条件的消息。
8. **消费进度**：消费者会维护一个消费进度（offset），记录已经消费到的消息位置。Broker根据这个进度来推送新的消息。
9. **故障恢复**：如果消费者发生故障，Broker会将未确认的消息重新分配给其他消费者，以确保消息的可靠性。

总之，RocketMQ采用拉模型（Pull Model）来处理消息传递，消费者主动从Broker拉取消息，而不是Broker推送给消费者。这种设计有助于提高系统的可扩展性和灵活性。

### 代码练习

主要基于SpringBoot整合RocketMQ。引入依赖

```xml
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-spring-boot-starter</artifactId>
    <version>2.2.2</version>
</dependency>
```

#### 生产者

```java
public class MQProducer {
	// 注入
    @Autowired
    private RocketMQTemplate rocketMQTemplate;

    public void sendMsg(String topic, Object body) {
        Message<Object> build = MessageBuilder.withPayload(body).build();// 构建消息
        rocketMQTemplate.send(topic, build);//发送
    }
}
```

#### 消费者

通过@RocketMQMessageListener注解，实现绑定消费者组和topic。

```java
@RocketMQMessageListener(consumerGroup = MQConstant.SEND_MSG_GROUP, topic = MQConstant.SEND_MSG_TOPIC)
@Component
public class MsgSendConsumer implements RocketMQListener<MsgSendMessageDTO> {//这里的泛型，能指定消费类型。
@Override
    public void onMessage(MsgSendMessageDTO dto) {
        Message message = messageDao.getById(dto.getMsgId());
        //逻辑处理。
    }
}
```

### 如何保证顺序消费消息

最常见的场景就是数据同步（从一个库同步到另一个库），当数据量大的时候数据同步压力也是很大的。这种情况我们都是怼到队列里面去，然后慢慢消费的，那问题就来了呀，我们在数据库同时对一个Id的数据进行了增、改、删三个操作，但是你消息发过去消费的时候变成了改，删、增，这样数据就不对了。本来一条数据应该删掉了，结果在你那儿却还在，这不是出大问题！通常我们所说的顺序消费消息指的是生产者按照顺序发送，消费者按照顺序进行消费，听起来简单，但做起来却非常困难。典型的要求消息顺序消费的场景就是基于binlog作数据同步。

无论是Kafka还是RocketMQ，每个主题下面都有若干分区（RocketMQ叫队列），如果消息被分配到不同的分区中，那么Kafka是不能保证消息的消费顺序的，因为每个分区都分配到一个消费者，此时无法保证消费者的消费先后，因此如果需要进行消息具有消费顺序性，可以在生产端指定这一类消息的key，这类消息都用相同的key进行消息发送，kafka就会根据key哈希取模选取其中一个分区进行存储，由于一个分区只能由一个消费者进行监听消费，因此这时候一个分区中的消息就具有消费的顺序性了。

**发送端**
但以上情况只是在正常情况下可以保证顺序消息，但发生故障后，就没办法保证消息的顺序了，我总结以下两点：

- 当生产端是异步发送时，此时有消息发送失败，比如你异步发送了1、2、3消息，2消息发送异常要重发，这时候顺序就乱了；
- 当部分Broker宕机，会触发Reblance，导致同一Topic下的分区数量有变化，此时生产端有可能会把顺序消息发送到不同的分区，这时会发生短暂消息顺序不一致的现象，如果生产端指定分区发送，则该分区所在的Broker宕机后将直接不可用；

针对以上两点，生产端必须保证单线程同步发送，这还好解决，针对第二点，想要做到严格的消息顺序，就要保证当集群出现故障后集群立马不可用，或者主题做成单分区，但这么做大大牺牲了集群的高可用，单分区也会令集群性能大大降低。

**存储端**
对于存储端，要保证消息顺序，会有以下几个问题：

- 要保证顺序的消息不能多分区存储，也就是只能放置在同一个分区中，在Kafka中，它叫做partition；在RocketMQ中，它叫做queue。 如果消息分散到多个分区里面，自然不能保证顺序；
- 即使顺序消息都存储在一个分区中，还会有第2个问题。Broker挂了之后，能否切换到其他副本机器？也就是高可用问题；

比如你当前的Broker挂了，上面还有消息没有消费完。此时切换到副本机器，可用性保证了，但消息顺序就乱掉了。

要想保证，一方面要同步复制不能异步复制；另1方面得保证，切机器之前，挂掉的机器上面，所有消息必须消费完了，不能有残留，很明显，这个很难！！！

**消费端**
对于消费端，不能并行消费，即不能开多线程或者多个客户端消费同1个队列。RocketMQ会为每个队列分配一个PullRequest，并将其放入pullRequestQueue，PullMessageService线程会不断轮询从pullRequestQueue中取出PullRequest去拉取消息，接着将拉取到的消息给到ConsumeMessageService处理，ConsumeMessageService有两个子接口：

```java
// 并发消息消费逻辑实现类
org.apache.rocketmq.client.impl.consumer.ConsumeMessageConcurrentlyService;
// 顺序消息消费逻辑实现类
org.apache.rocketmq.client.impl.consumer.ConsumeMessageOrderlyService;
```

ConsumeMessageConcurrentlyService内部有一个线程池，用于并发消费，同样地，如果需要顺序消费，那么RocketMQ提供了 ConsumeMessageOrderlyService类进行顺序消息消费处理。从ConsumeMessageOrderlyService源码中能够看出RocketMQ能够实现局部消费顺序，主要有以下两点：

- RocketMQ会为每个队列建一个对象锁，在消费某一个消息消费队列时先加锁，意味着一个消费者内消费线程池中的并发度是消费队列级别，同一个消费队列在同一时刻只会被一个线程消费，其他线程排队消费。保证了当前Consumer内，同一队列的消息进行串行消费。
- 向Broker端请求锁定当前顺序消费的队列，防止在消费过程中被分配给其它消费者处理从而打乱消费顺序。

从上面的分析可以看出，要保证消息的严格有序，有多么困难！发送端和接收端的问题，还好解决一点，限制异步发送，限制并行消费。但对于存储端，机器挂了之后，切换的问题，就很难解决了。你切换了，可能消息就会乱；你不切换，那就暂时不可用。这2者之间，就需要权衡了。结论是：无论RocketMQ还是Kafka，都不保证消息的严格有序消费！