---
title: Netty学习
date: 2023-2-25 10:02:51
tags: 学习笔记
categories: 学习笔记
---

# Netty入门

## 1. 概述

Netty是一个**高性能的Java网络应用程序框架**，可以帮助开发人员**快速和容易**地创建高性能、可靠的网络应用程序。Netty可以帮助开发人员构建强大的网络服务器和客户端应用程序，以便更容易地交换数据。Netty提供了一系列的功能，例如：支持多种协议，提供强大的编解码器，支持异步和同步请求处理以及一系列的可靠性机制。

## 2. 入门

### （1）加入依赖

```java
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
    <version>4.1.39.Final</version>
</dependency>
```

### （2）服务器端

```java
public class HelloServer {
    public static void main(String[] args) {
        // 1. 启动器
        new ServerBootstrap()
            // 2. 添加处理器组
            .group(new NioEventLoopGroup())
            // 3. 选择服务器的ServerSocketChannel的实现
            .channel(NioServerSocketChannel.class)
            // 4. 每个Channel的处理
            .childHandler(
                new ChannelInitializer<NioSocketChannel>() {
                    // 5. 子处理器的初始化
                    @Override
                    protected void initChannel(NioSocketChannel ch) throws Exception {
                        ch.pipeline().addLast(new StringDecoder());
                        ch.pipeline().addLast(new ChannelInboundHandlerAdapter() {
                            @Override
                            public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                                System.out.println(msg);
                            }
                        });
                    }
                }
            )
            .bind(8080);
    }
}
```

- ServerBootstrap 创建服务器启动类
- .group(new NioEventLoopGroup()) 创建处理器组，可以理解成`线程池+Selector`
- .channel(NioServerSocketChannel.class) 基于`NIO的实现`
- .childHandler 每个`Channel的处理方式`
- ChannelInitializer `具体的处理器`
- pipeline `业务工作流`

### （3）客户端

```java
public class HelloClient {
    public static void main(String[] args) throws InterruptedException, IOException {
        // 创建客户端
        ChannelFuture client = new Bootstrap()
            .group(new NioEventLoopGroup())
            .channel(NioSocketChannel.class)
            .handler(new ChannelInitializer<NioSocketChannel>() {
                @Override
                protected void initChannel(NioSocketChannel ch) throws Exception {
                    ch.pipeline().addLast(new StringEncoder());
                }
            })
            .connect(new InetSocketAddress("localhost", 8080))
            .sync();
        // 发送数据
        Channel channel = client.channel();
        channel.writeAndFlush("hello world");
        System.in.read();
    }
}
```

- ServerBootstrap 创建客户端启动类
- .group(new NioEventLoopGroup()) 创建处理器组，可以理解成`线程池+Selector`
- .channel(NioServerSocketChannel.class) 基于`NIO的实现`
- .handler 创建具体的`业务处理器`
- connect `进行连接`
- sync `等待连接建立完毕`

### （4）流程梳理

正确的观念

- 把 `channel` 理解为数据的通道
- 把 `msg` 理解为流动的数据，最开始输入是 ByteBuf，但经过 pipeline 的加工，会变成其它类型对象，最后输出又变成 ByteBuf
- 把`handler` 理解为数据的处理工序
  - 工序有多道，合在一起就是 pipeline，pipeline 负责发布事件（读、读取完成...）传播给每个 handler， handler 对自己感兴趣的事件进行处理（重写了相应事件处理方法）
  - handler 分 Inbound 和 Outbound 两类
- 把 `eventLoop` 理解为处理数据的工人
  - 工人可以管理多个 channel 的 io 操作，并且一旦工人负责了某个 channel，就要负责到底（绑定）
  - 工人既可以执行 io 操作，也可以进行任务处理，每位工人有任务队列，队列里可以堆放多个 channel 的待处理任务，任务分为普通任务、定时任务
  - 工人按照 pipeline 顺序，依次按照 handler 的规划（代码）处理数据，可以为每道工序指定不同的工人 

## 3. 组件

### （1）EventLoop

- EventLoop （事件循环对象）本质是一个`单线程执行器`（同时维护了一个 Selector），里面有 run 方法处理 Channel 上源源不断的 io 事件。
- EventLoopGroup `是一组 EventLoop`，Channel 一般会调用 EventLoopGroup 的 register 方法来绑定其中一个 EventLoop，后续这个 Channel 上的 io 事件都由此 EventLoop 来处理（保证了 io 事件处理时的线程安全）

**演示EventLoopGroup和使用EventLoop提交任务。**

```java
@Slf4j
public class EventLoopGroup {
    public static void main(String[] args) {
        // 内部创建了两个 EventLoop, 每个 EventLoop 维护一个线程
        NioEventLoopGroup group = new NioEventLoopGroup(2);
        System.out.println(group.next());
        System.out.println(group.next());
        System.out.println(group.next());

        // 一个EventLoop 进行 普通任务
        group.next().submit(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            log.debug("ok");
        });
        log.debug("main");
    }
}
```

- **EventLoop绑定机制**

一旦工人负责了某个 channel（一个Channel相当于一个链接），就要负责到底（绑定）

- 划分事件

划分事件：一个EventLoop专门处理Accept事件，其他EventLoop处理其他事件

```java
// 2. 一个EventLoop处理Accept事件，两个EventLoop处理其他事件
.group(new NioEventLoopGroup(1), new NioEventLoopGroup(2))
```

- Handler换人

Handler换人：在增加一个EventLoopGroup专门处理耗时长的事件

```java
@Slf4j
public class HandlerServer {
    public static void main(String[] args) {
        // 创建一个专门处理慢事件的EventLoop
        DefaultEventLoopGroup slowHandler = new DefaultEventLoopGroup(1);

        new ServerBootstrap()
            .group(new NioEventLoopGroup(1), new NioEventLoopGroup(2))
            .channel(NioServerSocketChannel.class)
            .childHandler(new ChannelInitializer<NioSocketChannel>() {
                @Override
                protected void initChannel(NioSocketChannel ch) throws Exception {
                    ch.pipeline().addLast(new StringDecoder());
                    ch.pipeline().addLast("handler1",new ChannelInboundHandlerAdapter(){
                        @Override
                        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                            log.debug("handler1---> {}",(String)  msg);
                            ctx.fireChannelRead(msg);
                        }
                    }).addLast(slowHandler, "slowHandler", new ChannelInboundHandlerAdapter(){
                        // 调用专用EventLoop进行处理
                        @Override
                        public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                            log.debug("slowHandler----> {}", (String) msg);
                        }
                    });
                }
            })
            .bind(8080);
    }
}
```

优雅关闭采用`shutdownGracefully` 方法。该方法会首先切换 `EventLoopGroup` 到关闭状态从而拒绝新的任务的加入，然后在任务队列的任务都处理完成后，停止线程的运行。从而确保整体应用是在正常有序的状态下退出的

### （2）Channel

channel 的主要功能

- close() 可以用来关闭 channel
- closeFuture() 处理关闭后的问题
  - sync 方法作用是同步等待 channel 关闭
  - 而 addListener 方法是异步等待 channel 关闭
- pipeline() 方法添加处理器
- write() 方法将数据写入
- writeAndFlush() 方法将数据写入并刷出
- **与服务器建立连接**

```java
ChannelFuture channelFuture = new Bootstrap()
    .group(new NioEventLoopGroup())
    .channel(NioSocketChannel.class)
    .handler(new ChannelInitializer<Channel>() {
        @Override
        protected void initChannel(Channel channel) throws Exception {
            channel.pipeline().addLast(new StringEncoder());
        }
    })
    .connect("127.0.0.1", 8080);

// 方法1
Channel channel = channelFuture.sync().channel();
channel.writeAndFlush("hello world");

// 方法2
channelFuture.addListener(new ChannelFutureListener() {
    @Override
    public void operationComplete(ChannelFuture channelFuture) throws Exception {
        channelFuture.channel().writeAndFlush("hello world");
    }
});
```

- 方法1：同步方法
- 方法2：异步方法

connect是异步方法，如果不使用sync进行阻塞，会拿到空的channel对象。

上述代码中用两种方法拿到真正的channel对象

- **采用异步方法，优雅的关闭EventLoopGroup**

```java
@Slf4j
public class CloseClient {
    public static void main(String[] args) throws InterruptedException {
        // 优雅的关闭nioEventLoopGroup
        NioEventLoopGroup group = new NioEventLoopGroup();

        Channel channel = new Bootstrap()
            .group(group)
            .channel(NioSocketChannel.class)
            .handler(new ChannelInitializer<NioSocketChannel>() {
                @Override
                protected void initChannel(NioSocketChannel nioSocketChannel) throws Exception {
                    nioSocketChannel.pipeline().addLast(new StringEncoder());
                }
            }).connect("localhost", 8080)
            .sync()
            .channel();

        new Thread(()-> {
            Scanner scanner = new Scanner(System.in);
            while(true) {
                String line = scanner.nextLine();
                if("q".equals(line)) {
                    // 错误做法：异步操作，可能会导致善后工作先执行，close后执行
                    channel.close();
                    // log.debug("处理善后工作");
                    break;
                }
                channel.writeAndFlush(line);
            }

        }).start();

        // 正确做法：等待善后工作结束
        channel.closeFuture().sync();
        log.debug("处理善后工作");
        group.shutdownGracefully();
    }
}
```

由于close是异步方法，如果不使用sync进行阻塞，可能直接就进行善后工作了。因此要先进行sync，然后再进行善后工作。

上面的代码中还讲到了优雅的关闭nioEventLoopGroup。

**异步提升的是什么**？ 简单来说就是让单线程最大限度地处理事件

- 单线程没法异步提高效率，必须配合多线程、多核 cpu 才能发挥异步的优势
- 异步并没有缩短响应时间，反而有所增加
- 合理进行任务拆分，也是利用异步的关键

### （3）Feature & Promise（了解）

在**异步处理**时，经常用到这两个接口

Netty的Future继承自JDK的Future，而Promise则扩展了Netty Future。

- jdk Future **只能同步等待任务结束**（或成功、或失败）才能得到结果
- netty Future 可以同步等待任务结束得到结果，也可以**异步方式得到结果**，但都是要等任务结束
- netty Promise 不仅有 netty Future 的功能，而且脱离了任务独立存在，**只作为两个线程间传递结果的容器**

| 功能/名称    | jdk Future                     | netty Future                                                 | Promise      |
| ------------ | ------------------------------ | ------------------------------------------------------------ | ------------ |
| cancel       | 取消任务                       | -                                                            | -            |
| isCanceled   | 任务是否取消                   | -                                                            | -            |
| isDone       | 任务是否完成，不能区分成功失败 | -                                                            | -            |
| get          | 获取任务结果，阻塞等待         | -                                                            | -            |
| getNow       | -                              | 获取任务结果，非阻塞，还未产生结果时返回 null                | -            |
| await        | -                              | 等待任务结束，如果任务失败，不会抛异常，而是通过 isSuccess 判断 | -            |
| sync         | -                              | 等待任务结束，如果任务失败，抛出异常                         | -            |
| isSuccess    | -                              | 判断任务是否成功                                             | -            |
| cause        | -                              | 获取失败信息，非阻塞，如果没有失败，返回null                 | -            |
| addLinstener | -                              | 添加回调，异步接收结果                                       | -            |
| setSuccess   | -                              | -                                                            | 设置成功结果 |
| setFailure   | -                              | -                                                            | 设置失败结果 |

```java
@Slf4j
public class TestFuture {
    @Test
    public void jdkFuture() throws ExecutionException, InterruptedException {
        // 线程池
        ExecutorService service = Executors.newFixedThreadPool(2);
        Future<Integer> future = service.submit(new Callable<Integer>() {
            @Override
            public Integer call() throws Exception {
                log.debug("进行计算");
                Thread.sleep(1000);
                return 50;
            }
        });

        log.debug("等待结果");
        log.debug("结果是：{}", future.get());
    }
    @Test
    public void nettyFuture() throws ExecutionException, InterruptedException {
        EventLoop eventLoop = new NioEventLoopGroup().next();
        io.netty.util.concurrent.Future<Integer> future = eventLoop.submit(new Callable<Integer>() {
            @Override
            public Integer call() throws Exception {
                log.debug("进行计算");
                return 70;
            }
        });
        log.debug("等待结果");
        log.debug("结果是：{}", future.get());
    }
    @Test
    public void nettyPromise() throws ExecutionException, InterruptedException {
        // 1. 创建eventLoop对象
        EventLoop eventLoop = new NioEventLoopGroup().next();
        // 2. 创建结果容器
        DefaultPromise<Integer> promise = new DefaultPromise<>(eventLoop);
        new Thread(() -> {
            log.debug("开始计算");
            try {
                // int i = 1 / 0;
                Thread.sleep(1000);
                promise.setSuccess(80);
            } catch (InterruptedException e) {
                e.printStackTrace();
                promise.setFailure(e);
            }
        }).start();
        log.debug("等待结果");
        log.debug("结果是：{}", promise.get());
    }
}
```

上面的代码演示了JDK的Future和Netty的Future、Promise的基本使用。

### （4）Pipeline & Handler（重点）

ChannelHandler 用来处理 Channel 上的各种事件，分为**入站、出站**两种。所有 ChannelHandler 被连成一串，就是 Pipeline

- 入站处理器通常是 ChannelInboundHandlerAdapter 的子类，**主要用来读取客户端数据**
- 出站处理器通常是 ChannelOutboundHandlerAdapter 的子类，**主要对写回结果进行加工**

```java
public static void main(String[] args) {
new ServerBootstrap()
    .group(new NioEventLoopGroup())
    .channel(NioServerSocketChannel.class)
    .childHandler(new ChannelInitializer<NioSocketChannel>() {
        protected void initChannel(NioSocketChannel ch) {
            ChannelPipeline pipeline = ch.pipeline();
            pipeline.addLast(new ChannelInboundHandlerAdapter(){
                @Override
                public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                    log.debug("1");;
                    super.channelRead(ctx, msg);
                }
            });
            pipeline.addLast(new ChannelInboundHandlerAdapter(){
                @Override
                public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                    log.debug("2");;
                    super.channelRead(ctx, msg);
                }
            });
            pipeline.addLast(new ChannelInboundHandlerAdapter(){
                @Override
                public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
                    log.debug("3");;
                    // 最后一个入栈处理器，不同继续传递了
                    // super.channelRead(ctx, msg);

                    // 必须使用ch.channel()进行写入数据
                    ch.writeAndFlush(ctx.alloc().buffer().writeBytes("hello".getBytes()));
                }
            });
            pipeline.addLast(new ChannelOutboundHandlerAdapter(){
                @Override
                public void write(ChannelHandlerContext ctx, Object msg,
                                  ChannelPromise promise) throws Exception {
                    log.debug("4");;
                    super.write(ctx, msg, promise);
                }
            });
            pipeline.addLast(new ChannelOutboundHandlerAdapter(){
                @Override
                public void write(ChannelHandlerContext ctx, Object msg,
                                  ChannelPromise promise) throws Exception {
                    log.debug("5");;
                    super.write(ctx, msg, promise);
                }
            });
            pipeline.addLast(new ChannelOutboundHandlerAdapter(){
                @Override
                public void write(ChannelHandlerContext ctx, Object msg,
                                  ChannelPromise promise) throws Exception {
                    log.debug("6");;
                    super.write(ctx, msg, promise);
                }
            });
        }
    })
    .bind(8080);
}
```

输出结果`1 2 3 6 5 4`

可以看到，`ChannelInboundHandlerAdapter 是按照 addLast 的顺序执行的`，而 `ChannelOutboundHandlerAdapter 是按照 addLast 的逆序执行的`。

触发读事件时，入站处理器按照**In1、In2、In3**的顺序依次执行

触发写事件时，出站处理器按照**Out1、Out2、Out3**的顺序依次执行。

**详细解释：**

- 入站处理器中，ctx.fireChannelRead(msg) 是 

  调用下一个入站处理器

  - 如果注释掉 1 处代码，则仅会打印 1
  - 如果注释掉 2 处代码，则仅会打印 1 2

- 3 处的 ctx.channel().write(msg) 会 

  从尾部开始触发

   后续出站处理器的执行

  - 如果注释掉 3 处代码，则仅会打印 1 2 3

- 类似的，出站处理器中，ctx.write(msg, promise) 的调用也会 

  触发上一个出站处理器

  - 如果注释掉 6 处代码，则仅会打印 1 2 3 6

- ctx.channel().write(msg) vs ctx.write(msg) （一个小细节，触发的出战处理器位置不一样）

  - 都是触发出站处理器的执行
  - **ctx.channel().write(msg) 从尾部开始查找出站处理器**
  - **ctx.write(msg) 是从当前节点找上一个出站处理器**
  - 3 处的 ctx.channel().write(msg) 如果改为 ctx.write(msg) 仅会打印 1 2 3，因为节点3 之前没有其它出站处理器了
  - 6 处的 ctx.write(msg, promise) 如果改为 ctx.channel().write(msg) 会打印 1 2 3 6 6 6... 因为 ctx.channel().write() 是从尾部开始查找，结果又是节点6 自己

- 快捷测试采用EmbeddedChannel channnel（了解）

```java
ChannelInboundHandlerAdapter h1 = new ChannelInboundHandlerAdapter(){
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println("h1");
        super.channelRead(ctx, msg);
    }
};
ChannelInboundHandlerAdapter h2 = new ChannelInboundHandlerAdapter(){
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println("h2");
        super.channelRead(ctx, msg);
    }
};
ChannelOutboundHandlerAdapter h3 = new ChannelOutboundHandlerAdapter() {
    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
        System.out.println("h3");
        super.write(ctx, msg, promise);
    }
};
ChannelOutboundHandlerAdapter h4 = new ChannelOutboundHandlerAdapter() {
    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
        System.out.println("h4");
        super.write(ctx, msg, promise);
    }
};
EmbeddedChannel channel = new EmbeddedChannel(h1, h2, h3, h4);
channel.writeOneInbound(ByteBufAllocator.DEFAULT.buffer().writeBytes("world".getBytes()));
```

### （5）ByteBuf

ByteBuf 是Netty中比较重要的类，它是Netty中用来存储数据的容器，它提供了一系列的方法来读写数据，比如 readInt() 、 writeInt() 、 readLong() 、 writeLong() 等，及一些索引类的操作等。**ByteBuf 是基于 ByteBuffer 来实现的，但是提供了更多的功能**。

ByteBuf 优势

- 池化 - 可以重用池中 ByteBuf 实例，更节约内存，减少内存溢出的可能
- 读写指针分离，不需要像 ByteBuffer 一样切换读写模式
- 可以自动扩容
- 支持链式调用，使用更流畅
- 很多地方体现零拷贝，例如 slice、duplicate、CompositeByteBuf

**1）创建**

```java
ByteBuf buf = ByteBufAllocator.DEFAULT.buffer();
buf.writeBytes("hello world".getBytes());
```

**2）直接内存 vs 堆内存**

```java
// 创建池化基于堆的 ByteBuf
ByteBuf buffer = ByteBufAllocator.DEFAULT.heapBuffer(10);
// 创建池化基于直接内存的 ByteBuf
ByteBuf buffer = ByteBufAllocator.DEFAULT.directBuffer(10);
```

- 直接内存创建和销毁的代价昂贵，但读写性能高（少一次内存复制），适合配合池化功能一起用
- 直接内存对 GC 压力小，因为这部分内存不受 JVM 垃圾回收的管理，但也要注意及时主动释放

**3）池化 vs 非池化**

池化的最大意义在于可以重用 ByteBuf，优点有

- 没有池化，则每次都得创建新的 ByteBuf 实例，这个操作对直接内存代价昂贵，就算是堆内存，也会增加 GC 压力
- 有了池化，则可以重用池中 ByteBuf 实例，并且采用了与 jemalloc 类似的内存分配算法提升分配效率
- 高并发时，池化功能更节约内存，减少内存溢出的可能
- 通过参数开启\关闭池化：`Dio.netty.allocator.type={unpooled|pooled}`

**4）写入**

- writeInt(int value)      写入 int 值
- writeByte(int value)   写入 byte 值
- writeBytes(byte[] src)  写入 byte[]

**5）扩容**

再写入一个 int 整数时，容量不够了（初始容量是 10），这时会引发扩容

扩容规则是：

- 如何写入后数据大小未超过 512，则选择下一个 16 的整数倍，例如写入后大小为 12 ，则扩容后 capacity 是 16
- 如果写入后数据大小超过 512，则选择下一个 2^n，例如写入后大小为 513，则扩容后 capacity 是 2^10=1024（2^9=512 已经不够了）
- 扩容不能超过 max capacity 会报错

**6）读取**

例如读了 4个字节。读过的内容，就属于废弃部分了，再读只能读那些尚未读取的部分。

但是如果还想读，可以在 read 前先做个标记 mark，然后再reset回去。

**7）retain & release（了解）**

由于 Netty 中有堆外内存的 ByteBuf 实现，**堆外内存最好是手动来释放**，而不是等 GC 垃圾回收。Netty 这里采用了引用计数法来控制回收内存，每个 ByteBuf 都实现了 ReferenceCounted 接口。

- 调用 release 方法计数减 1，如果计数为 0，ByteBuf 内存被回收
- 调用 retain 方法计数加 1，表示调用者没用完之前，其它 handler 即使调用了 release 也不会造成回收

**8）slice（重点了解）**

NIO中的零拷贝是指将FileChannel直接拷贝到SocketChannel，没有经过Java内存。避免了数据在用户态在内核态之间的拷贝，提高了数据传输的效率。 2. 而Netty零拷贝是将ByteBuf中的数据中的进行拷贝，有点浅拷贝的意思，即尽可能避免数据复制和内存拷贝，来达到零拷贝的效果。

【零拷贝】的体现之一，对原始 ByteBuf 进行切片成多个 ByteBuf，切片后的 ByteBuf 并没有发生内存复制，还是使用原始 ByteBuf 的内存，切片后的 ByteBuf 维护独立的 read，write 指针。

以下是一个使用 slice的实例代码，**有点浅拷贝的意思**

```java
ByteBuf buffer = ByteBufAllocator.DEFAULT.buffer();
buffer.writeBytes("hello world!".getBytes());

ByteBuf s1 = buffer.slice(0, 5);
ByteBuf s2 = buffer.slice(6,6);
s1.setByte(0, 'a');
log(s1);
log(s2);
log(buffer);
```

**9）duplicate**

【零拷贝】的体现之一，就好比截取了原始 ByteBuf 所有内容。

**10）copy：**会将底层内存数据进行深拷贝

**11）CompositeByteBuf：**【零拷贝】的体现之一，可以将多个 ByteBuf 合并为一个逻辑上的 ByteBuf，避免拷贝