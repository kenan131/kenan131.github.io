---
title: 基于消息头和消息体来解决socket半包问题
date: 2024-2-12
tags: JAVA
categories: JAVA
---

### socket介绍

#### 什么是粘包、半包

粘包：例如服务端依次将两条消息发送给客户端，我们暂且简单的将这两条消息举例为"Hello"、"JAVA"，而客户端一次性读取到的内容却是"HelloJAVA"，像这种一次性读取到两条消息中数据内容的情况称之为粘包。

半包：例如服务端发送消息"Hello"给客户端，而客户端依次读取到"Hel"，"lo"两条消息，这种情况称之为半包。

#### 粘包、半包发生的原因

粘包：消息发送方发送完完整的消息后，接收方没有及时处理(比如网络开小差，未能及时读取消息)，数据滞留于缓冲区，此时发送方又继续发送了其他消息，那么接收方下次在缓冲区读取时，一次性读取到大于一条消息数据造成粘包。

半包：发送方发送消息数据大小为512字节，而接收方缓冲区剩余已不足512字节，造成半包。究其根本原因，TCP为流式协议，消息不存在边界。

#### 解决方案

**1.固定长度法：**

- 服务端和客户端规定固定长度的缓冲区，当消息数据长度不足时，使用规定的填充字符进行填充。-
- 弊端：增加了不必要的数据传输，造成网络传输负担，不建议使用。

**2.结束标识法：**

- 在包体尾部增加标识符表示一条完整的消息数据已经结束。
- 弊端：若消息体本身包含该标识符需要做转义处理，因此效率依然不高。

**3.长度信息法：**

- 将包体分为消息头+消息体，消息头中信息为消息体的长度，接收方通过该长度信息读取后面指定长度的内容，需要注意的是需限制可能的最大长度从而规定长度占用字节数。该方法为处理粘包半包问题的常用方法。
- 实现起来较为复杂。

### 代码实现（消息头+消息体实现）

本实现方法采用封装消息头的方式实现，消息头固定为4个字节，消息头的内容为消息体的长度。发送数据时将数据报分为两段，即消息头 + 消息体。这样当读取事件触发的时候，先读取4个字节的固定消息头，再通过消息头的长度信息，去读取消息体，即可实现毡包半包的情况。

#### server端

```java
// 初始化
private ServerSocketChannel serverSocketChannel;
private Selector selector;
public void init() throws IOException {
    serverSocketChannel = ServerSocketChannel.open();
    //切换为非阻塞模式
    serverSocketChannel.configureBlocking(false);
    serverSocketChannel.bind(new InetSocketAddress(9000));
    selector = Selector.open();
    serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
}
```

```java
// server端具体执行逻辑
public void start()throws IOException {
    init();
    while(!Thread.currentThread().isInterrupted()){
        selector.select();
        //获取当前选择器中所有注册的监听事件
        for (Iterator<SelectionKey> it = selector.selectedKeys().iterator(); it.hasNext(); ) {
            SelectionKey key = it.next();
            //删除已选的key,以防重复处理
            it.remove();
            //如果"接收"事件已就绪
            if (key.isAcceptable()) {
                //交由接收事件的处理器处理
                handleAcceptRequest();
            } else if (key.isReadable()) {
                SocketChannel socketChannel = (SocketChannel) key.channel();
                // 将socketchannel交给读事件处理器处理
                handleRead(socketChannel);
            }
        }
    }
}
// 连接事件处理器
private void handleAcceptRequest() {
    try {
        System.out.println("连接成功！");
        SocketChannel client = serverSocketChannel.accept();
        // 接收的客户端也要切换为非阻塞模式
        client.configureBlocking(false);
        // 监控客户端的读操作是否就绪
        client.register(selector, SelectionKey.OP_READ);

    } catch (IOException e) {
        e.printStackTrace();
    }
}
// 读取事件处理器
private void handleRead(SocketChannel socketChannel)throws IOException{
    ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
    ByteBuffer buffer = ByteBuffer.allocate(1024);
    try{
        ByteBuffer fixBuffer = ByteBuffer.allocate(4); //固定头报文的4字节
        int len = socketChannel.read(fixBuffer);
        if (len <= 0 ){
            return ; // 如果长度小于0 则停止读取。
        }
        fixBuffer.flip();
        int messageLen = fixBuffer.getInt();  //一份数据包的长度
        System.out.println(messageLen + " 数据包大小");
        while(messageLen > 0 && (len = socketChannel.read(buffer)) != -1){
            buffer.flip();
            outputStream.write(buffer.array(),0,len);
            buffer.clear();
            messageLen = messageLen - 1024;
        }
        byte[] bytes = outputStream.toByteArray();
        outputStream.close();
        Message message = (Message) HessianSerializer.deserialize(bytes); // 序列化
        System.out.println(message.toString());//消息内容
    }catch (IOException e){
        System.out.println("socket 断开连接");
        socketChannel.close();
    }
}
```

启动服务端

```java
public static void main(String[] args) throws Exception {
    Client client = new Client();
    client.start();
}
```

#### client端

```java
// 初始化
private Selector selector;
private SocketChannel clientChannel;
ByteArrayOutputStream outputStream;
private ByteBuffer buffer;
private ByteBuffer fixBuffer;

public void init() throws IOException {
    selector = Selector.open();
    clientChannel = SocketChannel.open(new InetSocketAddress("127.0.0.1", 9000));
    //设置客户端为非阻塞模式
    clientChannel.configureBlocking(false);
    clientChannel.register(selector, SelectionKey.OP_READ);
    buffer = ByteBuffer.allocate(1024);
    fixBuffer = ByteBuffer.allocate(4);
    outputStream = new ByteArrayOutputStream();
}
```

```java
// 启动服务
public void start() throws IOException, InterruptedException {
    init();
    new Thread(new Runnable() {
        @Override
        public void run() {
            process();
        }
    }).start();
    //休眠一段时间，等服务端启动完毕 再发送数据
    Thread.sleep(2000);
    sendMessageTest();// 测试发送数据！
}
public void process() {
    while(!Thread.currentThread().isInterrupted()){
        try{
            selector.select();
            for(Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();iterator.hasNext();){
                SelectionKey key = iterator.next();
                iterator.remove();
                if(key.isReadable()){
                    SocketChannel channel = (SocketChannel)key.channel();
                    int len = channel.read(fixBuffer);
                    if (len == -1){
                        return ;
                    }
                    fixBuffer.flip(); // 切换读取模式
                    int messageLen = fixBuffer.getInt();  //一份数据包的长度
                    fixBuffer.clear(); // 公用的buffer得清理数据
                    while(messageLen > 0 && (len = channel.read(buffer)) != -1){
                        buffer.flip();
                        outputStream.write(buffer.array(),0,len);
                        buffer.clear();
                        messageLen = messageLen - 1024;
                    }
                    byte[] bytes = outputStream.toByteArray();
                    outputStream.close();
                    Message message = (Message) HessianSerializer.deserialize(bytes); // 序列化消息
                    System.out.println(message);
                }else if(key.isWritable()){
                    System.out.println("写事件！");
                    SocketChannel channel = (SocketChannel) key.channel();
                    ByteBuffer attachment = (ByteBuffer)key.attachment();
                    channel.write(attachment);
                    if(!attachment.hasRemaining()){
                        key.attach(null);
                        key.interestOps(0);
                    }
                }
            }
        }catch (IOException e){
            e.printStackTrace();
        }
    }
}
```

```java
// 发送数据测试
public void sendMessageTest() throws InterruptedException, IOException {
    for (int i = 0; i < 10; i++){
        Message message = new Message("小明",12);
        byte[] body = HessianSerializer.serialize(message);
        int len = body.length;
        fixBuffer.putInt(len);
        fixBuffer.flip();
        byte[] head = fixBuffer.array();
        ByteBuffer messageBuffer = ByteBuffer.allocate(head.length + body.length);
        System.out.println(head.length + body.length);
        messageBuffer.put(head);
        messageBuffer.put(body);
        messageBuffer.flip();
        clientChannel.write(messageBuffer);
        if(messageBuffer.hasRemaining()){
            System.out.println("socket缓存区中，写不完该数据包！");
            clientChannel.register(selector,SelectionKey.OP_WRITE,messageBuffer);
        }
        System.out.println("发送数据成功！");
        fixBuffer.clear();
        Thread.sleep(50); // 避免发送过快导致 缓冲区丢包
    }
}
```


