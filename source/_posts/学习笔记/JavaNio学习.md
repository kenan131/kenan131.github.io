---
title: JavaNio学习
date: 2023-2-14 16:08:14
tags: 学习笔记
categories: 学习笔记
---

# 1. NIO基础

NIO（New I/O或者Non-blocking I/O）是从Java 1.4开始引入的一种新的I/O编程方式，相对于传统的IO来说，NIO更加灵活、高效、可靠，能够更好地处理海量数据和高并发场景。简单来说就是：**并发能力强。**

## 1. 三大组件

### 1.1 Channel

Channel是数据传输的**双向通道，Stream要不就是读，要不就是写。Channel比Stream更加底层。**

常见的Channel有FileChannel、SocketChannel。

### Buffer

Buffer是用来**缓冲读写数据的。**

常见的Buffer有ByteBuffer、IntBuffer

### 1.3 Selector

以前的多线程服务器程序，一个线程对应一个Socket，只能适合连接数少的场景。而线程池版，阻塞模式下，只能处理一个Socket链接。

**selector 的作用就是配合一个线程来管理多个 channel。适合连接数特别多，但流量低的场景。**

调用 selector 的 select() 会阻塞直到 channel 发生了读写就绪事件，这些事件发生，select 方法就会返回这些事件交给 thread 来处理。

## 2 ByteBuffer详解

### （1） 正确使用方法

- 向 buffer 写入数据，例如调用 **channel.read(buffer)**
- 调用 flip() 切换至**读模式**
- 从 buffer 读取数据，例如调用 **buffer.get()**
- 调用 clear() 切换至**写模式**
- 重复 1~4 步骤

```java
// 1. 输入输出流
try(FileChannel channel = new FileInputStream("D:/Java/netty/src/test/resources/data.txt").getChannel()) {
    // 2. 准备缓冲区
    ByteBuffer buffer = ByteBuffer.allocate(10);
    while(true) {
        // 3. 从channel读取数据，读到buffer中去
        int len = channel.read(buffer);
        log.debug("读到的字节数 {}", len);
        if(len == -1) {
            break;
        }
        // 4. 切换buffer读模式，打印内容
        buffer.flip();
        while(buffer.hasRemaining()) {
            byte b = buffer.get();
            log.debug("实际字节 {}", (char)b);
        }
        // 切换回写模式
        buffer.clear();
    }
} catch (IOException e) {
    throw new RuntimeException(e);
}
```

输出

```java
10:39:03 [DEBUG] [main] c.i.n.ChannelDemo1 - 读到字节数：10
10:39:03 [DEBUG] [main] c.i.n.ChannelDemo1 - 1
10:39:03 [DEBUG] [main] c.i.n.ChannelDemo1 - 2
10:39:03 [DEBUG] [main] c.i.n.ChannelDemo1 - 3
10:39:03 [DEBUG] [main] c.i.n.ChannelDemo1 - 4
10:39:03 [DEBUG] [main] c.i.n.ChannelDemo1 - 5
10:39:03 [DEBUG] [main] c.i.n.ChannelDemo1 - 6
10:39:03 [DEBUG] [main] c.i.n.ChannelDemo1 - 7
10:39:03 [DEBUG] [main] c.i.n.ChannelDemo1 - 8
10:39:03 [DEBUG] [main] c.i.n.ChannelDemo1 - 9
10:39:03 [DEBUG] [main] c.i.n.ChannelDemo1 - 0
10:39:03 [DEBUG] [main] c.i.n.ChannelDemo1 - 读到字节数：4
10:39:03 [DEBUG] [main] c.i.n.ChannelDemo1 - a
10:39:03 [DEBUG] [main] c.i.n.ChannelDemo1 - b
10:39:03 [DEBUG] [main] c.i.n.ChannelDemo1 - c
10:39:03 [DEBUG] [main] c.i.n.ChannelDemo1 - d
10:39:03 [DEBUG] [main] c.i.n.ChannelDemo1 - 读到字节数：-1
```

### （2） ByteBuffer 结构

ByteBuffer 有以下重要属性

- capacity：buffer总共大小
- position：当前读取、写入位置
- limit：读取限制

写模式下，**position 是写入位置，limit 等于容量。**

filp后变成读模式，**position变成读取位置，limit变成读取限制。**

compact 方法，是把未读完的部分向前压缩，然后**切换至写模式。**

### （3） 调试查看内部结构

- 调试工具类，复制粘贴就行

```java
import io.netty.util.internal.StringUtil;

import java.nio.ByteBuffer;

import static io.netty.util.internal.MathUtil.isOutOfBounds;
import static io.netty.util.internal.StringUtil.NEWLINE;

public class ByteBufferUtil {
    private static final char[] BYTE2CHAR = new char[256];
    private static final char[] HEXDUMP_TABLE = new char[256 * 4];
    private static final String[] HEXPADDING = new String[16];
    private static final String[] HEXDUMP_ROWPREFIXES = new String[65536 >>> 4];
    private static final String[] BYTE2HEX = new String[256];
    private static final String[] BYTEPADDING = new String[16];

    static {
        final char[] DIGITS = "0123456789abcdef".toCharArray();
        for (int i = 0; i < 256; i++) {
            HEXDUMP_TABLE[i << 1] = DIGITS[i >>> 4 & 0x0F];
            HEXDUMP_TABLE[(i << 1) + 1] = DIGITS[i & 0x0F];
        }

        int i;

        // Generate the lookup table for hex dump paddings
        for (i = 0; i < HEXPADDING.length; i++) {
            int padding = HEXPADDING.length - i;
            StringBuilder buf = new StringBuilder(padding * 3);
            for (int j = 0; j < padding; j++) {
                buf.append("   ");
            }
            HEXPADDING[i] = buf.toString();
        }

        // Generate the lookup table for the start-offset header in each row (up to 64KiB).
        for (i = 0; i < HEXDUMP_ROWPREFIXES.length; i++) {
            StringBuilder buf = new StringBuilder(12);
            buf.append(NEWLINE);
            buf.append(Long.toHexString(i << 4 & 0xFFFFFFFFL | 0x100000000L));
            buf.setCharAt(buf.length() - 9, '|');
            buf.append('|');
            HEXDUMP_ROWPREFIXES[i] = buf.toString();
        }

        // Generate the lookup table for byte-to-hex-dump conversion
        for (i = 0; i < BYTE2HEX.length; i++) {
            BYTE2HEX[i] = ' ' + StringUtil.byteToHexStringPadded(i);
        }

        // Generate the lookup table for byte dump paddings
        for (i = 0; i < BYTEPADDING.length; i++) {
            int padding = BYTEPADDING.length - i;
            StringBuilder buf = new StringBuilder(padding);
            for (int j = 0; j < padding; j++) {
                buf.append(' ');
            }
            BYTEPADDING[i] = buf.toString();
        }

        // Generate the lookup table for byte-to-char conversion
        for (i = 0; i < BYTE2CHAR.length; i++) {
            if (i <= 0x1f || i >= 0x7f) {
                BYTE2CHAR[i] = '.';
            } else {
                BYTE2CHAR[i] = (char) i;
            }
        }
    }

    /**
     * 打印所有内容
     * @param buffer
     */
    public static void debugAll(ByteBuffer buffer) {
        int oldlimit = buffer.limit();
        buffer.limit(buffer.capacity());
        StringBuilder origin = new StringBuilder(256);
        appendPrettyHexDump(origin, buffer, 0, buffer.capacity());
        System.out.println("+--------+-------------------- all ------------------------+----------------+");
        System.out.printf("position: [%d], limit: [%d]\\n", buffer.position(), oldlimit);
        System.out.println(origin);
        buffer.limit(oldlimit);
    }

    /**
     * 打印可读取内容
     * @param buffer
     */
    public static void debugRead(ByteBuffer buffer) {
        StringBuilder builder = new StringBuilder(256);
        appendPrettyHexDump(builder, buffer, buffer.position(), buffer.limit() - buffer.position());
        System.out.println("+--------+-------------------- read -----------------------+----------------+");
        System.out.printf("position: [%d], limit: [%d]\\n", buffer.position(), buffer.limit());
        System.out.println(builder);
    }

    public static void appendPrettyHexDump(StringBuilder dump, ByteBuffer buf, int offset, int length) {
        if (isOutOfBounds(offset, length, buf.capacity())) {
            throw new IndexOutOfBoundsException(
                    "expected: " + "0 <= offset(" + offset + ") <= offset + length(" + length
                            + ") <= " + "buf.capacity(" + buf.capacity() + ')');
        }
        if (length == 0) {
            return;
        }
        dump.append(
                "         +-------------------------------------------------+" +
                        NEWLINE + "         |  0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f |" +
                        NEWLINE + "+--------+-------------------------------------------------+----------------+");

        final int startIndex = offset;
        final int fullRows = length >>> 4;
        final int remainder = length & 0xF;

        // Dump the rows which have 16 bytes.
        for (int row = 0; row < fullRows; row++) {
            int rowStartIndex = (row << 4) + startIndex;

            // Per-row prefix.
            appendHexDumpRowPrefix(dump, row, rowStartIndex);

            // Hex dump
            int rowEndIndex = rowStartIndex + 16;
            for (int j = rowStartIndex; j < rowEndIndex; j++) {
                dump.append(BYTE2HEX[getUnsignedByte(buf, j)]);
            }
            dump.append(" |");

            // ASCII dump
            for (int j = rowStartIndex; j < rowEndIndex; j++) {
                dump.append(BYTE2CHAR[getUnsignedByte(buf, j)]);
            }
            dump.append('|');
        }

        // Dump the last row which has less than 16 bytes.
        if (remainder != 0) {
            int rowStartIndex = (fullRows << 4) + startIndex;
            appendHexDumpRowPrefix(dump, fullRows, rowStartIndex);

            // Hex dump
            int rowEndIndex = rowStartIndex + remainder;
            for (int j = rowStartIndex; j < rowEndIndex; j++) {
                dump.append(BYTE2HEX[getUnsignedByte(buf, j)]);
            }
            dump.append(HEXPADDING[remainder]);
            dump.append(" |");

            // Ascii dump
            for (int j = rowStartIndex; j < rowEndIndex; j++) {
                dump.append(BYTE2CHAR[getUnsignedByte(buf, j)]);
            }
            dump.append(BYTEPADDING[remainder]);
            dump.append('|');
        }

        dump.append(NEWLINE +
                "+--------+-------------------------------------------------+----------------+");
    }

    public static void appendHexDumpRowPrefix(StringBuilder dump, int row, int rowStartIndex) {
        if (row < HEXDUMP_ROWPREFIXES.length) {
            dump.append(HEXDUMP_ROWPREFIXES[row]);
        } else {
            dump.append(NEWLINE);
            dump.append(Long.toHexString(rowStartIndex & 0xFFFFFFFFL | 0x100000000L));
            dump.setCharAt(dump.length() - 9, '|');
            dump.append('|');
        }
    }

    public static short getUnsignedByte(ByteBuffer buffer, int index) {
        return (short) (buffer.get(index) & 0xFF);
    }
}
```

- 查看pos，limit位置，通过调用debugAll方法，我们可以很好的看到pos，limit位置

```java
// 分配完，默认是写模式
ByteBuffer buffer = ByteBuffer.allocate(10);
buffer.put((byte) 0x61);
// pos = 1, limit = 10
debugAll(buffer);

buffer.put(new byte[]{0x62, 0x63, 0x64});
// pos = 4, limit = 10
debugAll(buffer);

// 切换读模式
buffer.flip();
System.out.println(buffer.get());
// pos = 1, limit = 4
debugAll(buffer);
```

### （4）常用方法

- 分配空间

```java
// class java.nio.HeapByteBuffer    - java 堆内存，读写效果低，受到GC影响
System.out.println(ByteBuffer.allocate(16).getClass());
// class java.nio.DirectByteBuffer  - 直接内存，读写效率高（少一次拷贝），不会受GC影响。系统内存分配效率低，可能内存泄露
System.out.println(ByteBuffer.allocateDirect(16).getClass());
```

- 向 buffer 写入数据

```java
int readBytes = channel.read(buf);
buf.put((byte)127);
```

- 从 buffer 读取数据

```java
int writeBytes = channel.write(buf);
byte b = buf.get();
```

get 方法会让 position 读指针向后走，如果想重复读取数据

- 可以调用 rewind 方法将 position 重新置为 0
- 或者调用 get(int i) 方法获取索引 i 的内容，它不会移动读指针
- mark 和 reset（了解）：
  - mark 是在读取时，做一个标记，即使 position 改变，只要调用 reset 就能回到 mark 的位置
- **字符串与 ByteBuffer 互转**

```java
// 1. 简单转换为ByteBuffer -> 写模式
ByteBuffer buffer0 = ByteBuffer.allocate(16);
buffer0.put("hello".getBytes());

// 切换读模式
buffer0.flip();
String s0 = StandardCharsets.UTF_8.decode(buffer0).toString();
System.out.println(s0);

// 2. encode -> 读模式
ByteBuffer buffer1 = StandardCharsets.UTF_8.encode("world");
String s1 = StandardCharsets.UTF_8.decode(buffer1).toString();
System.out.println(s1);

// 3. warp
ByteBuffer buffer2 = ByteBuffer.wrap("hello".getBytes());
String s2 = StandardCharsets.UTF_8.decode(buffer2).toString();
System.out.println(s2);
```

### （5）分散读集中写

- 分散读：把一个Channel读取到三个Buffer当中去，减少数据的复制

```java
String baseUrl = "D:/Java/netty/src/test/resources/word.txt";
try (FileChannel channel = new RandomAccessFile(baseUrl, "rw").getChannel()) {
    ByteBuffer a = ByteBuffer.allocate(3);
    ByteBuffer b = ByteBuffer.allocate(3);
    ByteBuffer c = ByteBuffer.allocate(5);
    channel.read(new ByteBuffer[]{a, b, c});
    a.flip();
    b.flip();
    c.flip();
    debugAll(a);
    debugAll(b);
    debugAll(c);
} catch (IOException e) {
}
```

- 集中写：三个Buffer写到一个Channel里面去，减少数据的复制

```java
String baseUrl = "D:/Java/netty/src/test/resources/word.txt";
try (FileChannel channel = new RandomAccessFile(baseUrl, "rw").getChannel()) {
    ByteBuffer d = ByteBuffer.allocate(4);
    ByteBuffer e = ByteBuffer.allocate(4);
    d.put(new byte[]{'f', 'o', 'u', 'r'});
    e.put(new byte[]{'f', 'i', 'v', 'e'});
    d.flip();
    e.flip();
    debugAll(d);
    debugAll(e);
    channel.write(new ByteBuffer[]{d, e});
} catch (IOException e) {
}
```

### （6）初识粘包

网络上有多条数据发送给服务端，数据之间使用 \n 进行分隔但由于某种原因这些数据在接收时，被进行了重新组合，例如原始数据有3条为

- Hello,world\n
- I'm zhangsan\n
- How are you?\n

**变成了下面的两个 byteBuffer (黏包，半包)**

- Hello,world\nI'm zhangsan\nHo
- w are you?\n

现在要求你编写程序，将错乱的数据恢复成原始的按 \n 分隔的数据

**思路就是一个字符一个字符的比较**

```java
public static void main(String[] args) {
    ByteBuffer source = ByteBuffer.allocate(32);
    source.put("Hello,world\\nI'm zhangsan\\nHo".getBytes());
    split(source);

    source.put("w are you?\\nhaha!\\n".getBytes());
    split(source);
}

private static void split(ByteBuffer source) {
    source.flip();
    ByteBuffer target = ByteBuffer.allocate(15);
    for(int i = 0; i < source.limit(); i++) {
        if(source.get(i) == '\\n') {
            // 长度处理很关键
            int length = i + 1 - source.position();
            for(int j = 0; j < length; j++) {
                target.put(source.get());
            }
						// 打印字符
            debugAll(target);
            target.clear();
        }
    }
    source.compact();
}
```

## 3 文件编程（了解）

### （1）FileChannel

<aside> 💡 FileChannel 只能工作在阻塞模式下

</aside>

不能直接打开 FileChannel，必须通过 FileInputStream、FileOutputStream 或者 RandomAccessFile 来获取 FileChannel，它们都有 getChannel 方法

1. 通过 FileInputStream 获取的 channel 只能读
2. 通过 FileOutputStream 获取的 channel 只能写
3. 通过 RandomAccessFile 是否能读写根据构造 RandomAccessFile 时的读写模式决定

- **读取**

会从 channel 读取数据填充 ByteBuffer，返回值表示读到了多少字节，-1 表示到达了文件的末尾

```java
int readBytes = channel.read(buffer);
```

- **写入**

写入的正确姿势如下， SocketChannel

```java
ByteBuffer buffer = ...;
buffer.put(...); // 存入数据
buffer.flip();   // 切换读模式

while(buffer.hasRemaining()) {
    channel.write(buffer);
}
```

在 while 中调用 channel.write 是因为 write 方法并不能保证一次将 buffer 中的内容全部写入 channel

- **关闭**

channel 必须关闭，不过调用了 FileInputStream、FileOutputStream 或者 RandomAccessFile 的 close 方法会间接地调用 channel 的 close 方法

- **大小**

使用 size 方法获取文件的大小

- **强制写入**

操作系统出于性能的考虑，会将数据缓存，不是立刻写入磁盘。可以调用 force(true) 方法将文件内容和元数据（文件的权限等信息）立刻写入磁盘

### （2）**两个 Channel 传输数据（有用）**

- 小文件

  ```java
  String FROM = "helloword/data.txt";
  String TO = "helloword/to.txt";
  long start = System.nanoTime();
  try (FileChannel from = new FileInputStream(FROM).getChannel();
       FileChannel to = new FileOutputStream(TO).getChannel();
  ) {
      from.transferTo(0, from.size(), to);
  } catch (IOException e) {
      e.printStackTrace();
  }
  long end = System.nanoTime();
  System.out.println("transferTo 用时：" + (end - start) / 1000_000.0);
  ```

- 超大文件

  ```java
  public static void main(String[] args) {
      try (
              FileChannel from = new FileInputStream("data.txt").getChannel();
              FileChannel to = new FileOutputStream("to.txt").getChannel();
      ) {
          // 效率高，底层会利用操作系统的零拷贝进行优化
          long size = from.size();
          // left 变量代表还剩余多少字节
          for (long left = size; left > 0; ) {
              System.out.println("position:" + (size - left) + " left:" + left);
              left -= from.transferTo((size - left), left, to);
          }
      } catch (IOException e) {
          e.printStackTrace();
      }
  }
  ```

### （3）Path

- 遍历文件夹

```java
// 要遍历的文件夹
Path path = Paths.get("D:\\\\Java\\\\netty");
// 文件夹个数
AtomicInteger dirCount = new AtomicInteger();
// 文件个数
AtomicInteger fileCount = new AtomicInteger();
// 开始遍历
Files.walkFileTree(path, new SimpleFileVisitor<Path>(){
		// 进入文件夹之前的操作
    @Override
    public FileVisitResult preVisitDirectory(Path dir, BasicFileAttributes attrs) throws IOException {
        System.out.println("====> " + dir);
        dirCount.incrementAndGet();
        return super.preVisitDirectory(dir, attrs);
    }
		// 遍历到文件的操作
    @Override
    public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) throws IOException {
        System.out.println(file);
        fileCount.incrementAndGet();
        return super.visitFile(file, attrs);
    }
});
System.out.println(dirCount);
System.out.println(fileCount);
```

### （4） Files

检查文件是否存在

```java
Path path = Paths.get("helloword/data.txt");
System.out.println(Files.exists(path));
```

创建单级目录

```java
Path path = Paths.get("helloword/d1");
Files.createDirectory(path);
```

拷贝文件

```java
Path source = Paths.get("helloword/data.txt");
Path target = Paths.get("helloword/target.txt");

Files.copy(source, target);
```

## 4. 网络编程（重要）

### （1）非阻塞 vs 阻塞

**阻塞**

- 阻塞模式下，相关方法都会导致线程暂停
  - ServerSocketChannel.accept 会在没有连接建立时让线程暂停
  - SocketChannel.read 会在没有数据可读时让线程暂停
  - 阻塞的表现其实就是线程暂停了，暂停期间不会占用 cpu，但线程相当于闲置
- 单线程下，阻塞方法之间相互影响，几乎不能正常工作，需要多线程支持
- 但多线程下，有新的问题，体现在以下方面
  - 32 位 jvm 一个线程 320k，64 位 jvm 一个线程 1024k，如果连接数过多，必然导致 OOM，并且线程太多，反而会因为频繁上下文切换导致性能降低
  - 可以采用线程池技术来减少线程数和线程上下文切换，但治标不治本

**阻塞简单例子：问题，当连接A建立后，1s后，A发送数据服务器收不到数据。原因就是服务器还在等待另外一个客户端的lian**

- **服务器端**

```java
// 0. 创建buffer
ByteBuffer buffer = ByteBuffer.allocate(16);
// 1. 创建服务器
ServerSocketChannel ssc = ServerSocketChannel.open();
// 2. 绑定端口
ssc.bind(new InetSocketAddress(8080));

// 3. 连接集合
ArrayList<SocketChannel> channels = new ArrayList<>();

while(true) {
    log.debug("connecting...");
    SocketChannel sc = ssc.accept();
    log.debug("connect... {}", sc);
    channels.add(sc);
    for(SocketChannel channel: channels) {
        // 5. 接收客户端发送的数据
        log.debug("before read... {}", channel);
        channel.read(buffer); // 阻塞方法，线程停止运行
        buffer.flip();
        debugRead(buffer);
        buffer.clear();
        log.debug("after read...{}", channel);
    }
}
```

- **客户端**

```java
SocketChannel sc = SocketChannel.open();
sc.connect(new InetSocketAddress("localhost", 8080));
sc.write(Charset.defaultCharset().encode("1237\\n"));
sc.write(Charset.defaultCharset().encode("1234567890abc\\n"));
System.out.println("waiting...");
System.in.read();
```

**非阻塞**

- 非阻塞模式下，相关方法都不会让线程暂停。
  - accept 返回空，继续运行
  - read 返回0，继续运行
  - 写数据就直接写入，不需要等待网络发送数据
- 但非阻塞模式下，即使没有连接建立，和可读数据，线程仍然在不断运行，白白浪费了 cpu
- 数据复制过程中，线程实际还是阻塞的（AIO 改进的**地方）**

**非阻塞简单例子：服务器端，客户端代码不变**

其实代码主要就是多了：`ssc.configureBlocking(false);`

```java
// 0. 创建buffer
ByteBuffer buffer = ByteBuffer.allocate(16);

// 1. 创建服务器
ServerSocketChannel ssc = ServerSocketChannel.open();
// 非阻塞模式
ssc.configureBlocking(false);
// 2. 绑定端口
ssc.bind(new InetSocketAddress(8080));

// 3. 连接集合
ArrayList<SocketChannel> channels = new ArrayList<>();
while(true) {
    log.debug("connecting...");
		// 4. 进行连接
    SocketChannel sc = ssc.accept();
    if(sc != null) {
        sc.configureBlocking(false);
        channels.add(sc);
    }
    log.debug("connect... {}", sc);
    for(SocketChannel channel: channels) {
        // 5. 接收客户端发送的数据
        log.debug("before read... {}", channel);
        int len = channel.read(buffer); // 阻塞方法，线程停止运行
        if(len > 0) {
            buffer.flip();
            debugRead(buffer);
            buffer.clear();
        }
        log.debug("after read...{}", channel);
    }
}
```

**多路复用**

单线程可以配合 Selector 完成对多个 Channel 可读写事件的监控，这称之为多路复用

- 多路复用仅针对网络 IO，文件IO没有办法多路复用
- 如果不用 Selector 的非阻塞模式，线程大部分时间都在做无用功，而 Selector 能够保证
  - 有可连接事件时才去连接
  - 有可读事件才去读取
  - 有可写事件才去写入

### （2）Selector

- 好处：
  - 一个线程配合 selector 就可以监控多个 channel 的事件，事件发生线程才去处理。避免非阻塞模式下所做无用功
  - 让这个线程能够被充分利用
  - 节约了线程的数量
  - 减少了线程上下文切换
- 创建

```java
Selector selector = Selector.open();
```

- 绑定（注册）Channel事件（非常重要）
  - channel 必须工作在非阻塞模式
  - FileChannel 没有非阻塞模式，因此不能配合 selector 一起使用
  - 绑定的事件类型可以有
    - connect - 客户端连接成功时触发
    - accept - 服务器端成功接受连接时触发
    - read - 数据可读入时触发，有因为接收能力弱，数据暂不能读入的情况
    - write - 数据可写出时触发，有因为发送能力弱，数据暂不能写出的情况

```java
channel.configureBlocking(false);
SelectionKey key = channel.register(selector, 绑定事件);
```

- **监听 Channel 事件（非常重要）**

可以通过下面三种方法来监听是否有事件发生，方法的返回值代表有多少 channel 发生了事件

阻塞直到绑定事件发生**（超级常用）**

```java
int count = selector.select();
```

select 何时不阻塞：

- 主要是，事件发生时
- 调用 selector.wakeup()
- 调用 selector.close()
- selector 所在线程 interrupt

### （3）处理Accept事件（最简单的Selector使用）

**客户端代码不变，服务器代码如下：**

```java
import lombok.extern.slf4j.Slf4j;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.SocketException;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;

import static utils.ByteBufferUtil.debugRead;

@Slf4j
public class C2_Server {
    public static void main(String[] args) throws IOException {
        ServerSocketChannel ssc = ServerSocketChannel.open();
        ssc.bind(new InetSocketAddress(8080));
        ssc.configureBlocking(false);

        // 1. 创建Selector
        Selector selector = Selector.open();
        // 1. 注册Selector事件
        SelectionKey sscKey = ssc.register(selector, 0, null);
        sscKey.interestOps(SelectionKey.OP_ACCEPT);

        List<ServerSocketChannel> channels = new ArrayList<>();
        while(true) {
            // 2. select 方法
            selector.select();

            // 3. 处理事件
            Iterator<SelectionKey> iter = selector.selectedKeys().iterator();
            while(iter.hasNext()) {
		            SelectionKey key = iter.next();

		            // 4. 处理accept事件
		            ServerSocketChannel channel = (ServerSocketChannel) key.channel();
		            log.debug("key: {}", key);
		            SocketChannel sc = channel.accept();
		            log.debug("sc: {}", sc);
            }
        }
    }
}
```

<aside> 💡  事件发生后能否不处理

> 事件发生后，要么处理，要么取消（cancel），不能什么都不做，否则下次该事件仍会触发，这是因为 nio 底层使用的是水平触发

</aside>

### （4） 处理读事件

**客户端代码不变，服务器代码如下，当有可读事件时，自动向下执行。**

```java
ServerSocketChannel ssc = ServerSocketChannel.open();
ssc.bind(new InetSocketAddress(8080));
ssc.configureBlocking(false);

// 1. 注册channel
Selector selector = Selector.open();
SelectionKey sscKey = ssc.register(selector, 0, null);
sscKey.interestOps(SelectionKey.OP_ACCEPT);

List<ServerSocketChannel> channels = new ArrayList<>();
while(true) {
    // 2. select 方法
    selector.select();

    // 3. 处理事件
    Iterator<SelectionKey> iter = selector.selectedKeys().iterator();
    while(iter.hasNext()) {
        SelectionKey key = iter.next();
        // 必须要移除这个事件
        iter.remove();

        if(key.isAcceptable()) {
            // 处理accept事件
            ServerSocketChannel channel = (ServerSocketChannel) key.channel();
            log.debug("key: {}", key);
            SocketChannel sc = channel.accept();
            sc.configureBlocking(false);
            SelectionKey scKey = sc.register(selector, 0, null);
            scKey.interestOps(SelectionKey.OP_READ);
            log.debug("sc: {}", sc);
        } else if(key.isReadable()) {
            // 处理read事件
            try {
                ByteBuffer buffer = ByteBuffer.allocate(16);
                SocketChannel channel = (SocketChannel) key.channel();
                int len = channel.read(buffer);

                if(len == -1) {
                    key.cancel();
                    System.out.println("主动断开连接");
                } else {
                    buffer.flip();
                    debugRead(buffer);
                }
            } catch (SocketException e) {
                e.printStackTrace();
                key.cancel();
                System.out.println("强制断开连接");
            }
        }
    }
}
```

<aside> 💡 为何要 iter.remove()

> 因为 select 在事件发生后，就会将相关的 key 放入 selectedKeys 集合，但不会在处理完后从 selectedKeys 集合中移除，需要我们自己编码删除。例如
>
> - 第一次触发了 ssckey 上的 accept 事件，没有移除 ssckey
> - 第二次触发了 sckey 上的 read 事件，但这时 selectedKeys 中还有上次的 ssckey ，在处理时因为没有真正的 serverSocket 连上了，就会导致空指针异常 </aside>

### （5）处理消息边界

- 一种思路是固定消息长度，数据包大小一样，服务器按预定长度读取，缺点是浪费带宽
- 另一种思路是按分隔符拆分，缺点是效率低
- TLV 格式，即 Type 类型、Length 长度、Value 数据，类型和长度已知的情况下，就可以方便获取消息大小，分配合适的 buffer，缺点是 buffer 需要提前分配，如果内容过大，则影响 server 吞吐量
  - Http 1.1 是 TLV 格式
  - Http 2.0 是 LTV 格式

**在处理读事件的基础上，如果当前的Buffer大小不能存储完整的一条数据，就进行扩容Buffer。**

```java
public static void main(String[] args) throws IOException {
    ServerSocketChannel ssc = ServerSocketChannel.open();
    ssc.bind(new InetSocketAddress(8080));
    ssc.configureBlocking(false);

    // 1. 注册channel
    Selector selector = Selector.open();
    SelectionKey sscKey = ssc.register(selector, 0, null);
    sscKey.interestOps(SelectionKey.OP_ACCEPT);

    List<ServerSocketChannel> channels = new ArrayList<>();
    while(true) {
        // 2. select 方法
        selector.select();

        // 3. 处理事件
        Iterator<SelectionKey> iter = selector.selectedKeys().iterator();
        while(iter.hasNext()) {
            SelectionKey key = iter.next();
            iter.remove();

            if(key.isAcceptable()) {
                // 处理accept事件
                ServerSocketChannel channel = (ServerSocketChannel) key.channel();
                log.debug("key: {}", key);
                SocketChannel sc = channel.accept();
                sc.configureBlocking(false);

                ByteBuffer buffer = ByteBuffer.allocate(8);
                SelectionKey scKey = sc.register(selector, 0, buffer);
                scKey.interestOps(SelectionKey.OP_READ);
                log.debug("sc: {}", sc);
            } else if(key.isReadable()) {
                // 处理read事件
                try {
                    SocketChannel channel = (SocketChannel) key.channel();
                    ByteBuffer buffer = (ByteBuffer) key.attachment();
                    int len = channel.read(buffer);
                    System.out.println(len);

                    if(len == -1) {
                        key.cancel();
                        System.out.println("主动断开连接");
                    } else {
                        split(buffer);
                        if(buffer.position() == buffer.limit()) {
                            ByteBuffer newBuffer = ByteBuffer.allocate(buffer.capacity() * 2);
                            buffer.flip();
                            newBuffer.put(buffer);
                            key.attach(newBuffer);
                        }
                    }
                } catch (SocketException e) {
                    e.printStackTrace();
                    key.cancel();
                    System.out.println("强制断开连接");
                }
            }
        }
    }
}
private static void split(ByteBuffer source) {
    source.flip();
    for (int i = 0; i < source.limit(); i++) {
        // 找到一条完整消息
        if (source.get(i) == '\\n') {
            int length = i + 1 - source.position();
            // 把这条完整消息存入新的 ByteBuffer
            ByteBuffer target = ByteBuffer.allocate(length);
            // 从 source 读，向 target 写
            for (int j = 0; j < length; j++) {
                target.put(source.get());
            }
            debugAll(target);
        }
    }
    source.compact(); // 0123456789abcdef  position 16 limit 16
}
```

## 5. NIO vs BIO

Non-Blocking IO vs Blocking IO

### （1）Stream vs channel

- stream 仅支持阻塞 API，channel 同时支持阻塞、非阻塞 API，网络 channel 可配合 selector 实现多路复用
- 二者均为全双工，即读写可以同时进行
- stream 不会自动缓冲数据，channel 会利用系统提供的发送缓冲区、接收缓冲区（更为底层）

### （2）IO模型

[第1章_47_nio-概念剖析-io模型-阻塞非阻塞_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1py4y1E7oA?p=48&vd_source=37f4fe625ebf4a33ec516d862b747925)

同步阻塞、同步非阻塞、同步多路复用、异步IO

- 同步：线程自己去获取结果（一个线程）
- 异步：线程自己不去获取结果，而是由其它线程送结果（至少两个线程）

当调用一次 channel.read 或 stream.read 后，会切换至操作系统内核态来完成真正数据读取，而读取又分为两个阶段，分别为：

- 等待数据阶段
- 复制数据阶段

### （3）零拷贝

- 传统 IO 问题：传统的 IO 将一个文件通过 socket 写出

```java
File f = new File("helloword/data.txt");
RandomAccessFile file = new RandomAccessFile(file, "r");

byte[] buf = new byte[(int)f.length()];
file.read(buf);

Socket socket = ...;
socket.getOutputStream().write(buf);
```

内部工作流程是这样的：

1、java 本身并不具备 IO 读写能力，因此 read 方法调用后，要从 java 程序的**用户态**切换至**内核态**，去调用操作系统（Kernel）的读能力，将数据读入**内核缓冲区**。这期间用户线程阻塞，操作系统使用 **DMA**（Direct Memory Access）来实现文件读，其间也不会使用 cpu

> DMA 也可以理解为硬件单元，用来解放 cpu 完成文件 IO

2、程从**内核态**切换回**用户态**，将数据从**内核缓冲区**读入**用户缓冲区**（即 byte[] buf），这期间 cpu 会参与拷贝，无法利用 DMA

3、调用 write 方法，这时将数据从**用户缓冲区**（byte[] buf）写入 **socket 缓冲区**，cpu 会参与拷贝

4、接下来要向网卡写数据，这项能力 java 又不具备，因此又得从**用户态**切换至**内核态**，调用操作系统的写能力，使用 DMA 将 **socket 缓冲区**的数据写入网卡，不会使用 cpu

> 磁盘和内核缓冲区交互采用DMA，内核态和用户态交互采用CPU

用户态与内核态的切换发生了 3 次，这个操作比较重量级。数据拷贝了共 4 次

- **NIO优化**

通过 DirectByteBuf，将堆外内存映射到 jvm 内存中来直接访问使用。减少了一次数据拷贝，用户态与内核态的切换次数没有减少

- **进一步优化（linux 2.4）**

- java 调用 transferTo 方法后，要从 java 程序的**用户态**切换至**内核态**，使用 DMA将数据读入**内核缓冲区**，不会使用 cpu
- 只会将一些 offset 和 length 信息拷入 **socket 缓冲区**，几乎无消耗
- 使用 DMA 将 **内核缓冲区**的数据写入网卡，不会使用 cpu

整个过程仅只发生了一次用户态与内核态的切换，**数据拷贝了 2 次。**所谓的【零拷贝】，并不是真正无拷贝，而是在不会拷贝重复数据到 jvm 内存中，**零拷贝的优点有**

- 更少的用户态与内核态的切换
- 不利用 cpu 计算，减少 cpu 缓存伪共享
- 零拷贝适合小文件传输