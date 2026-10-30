---
知识点: Java IO 与 NIO
层级: 进阶
适用版本: JDK 8+（NIO.2 为 JDK 7 引入）
更新日期: 2026-10-31
关联:
  - Java
  - 计算机网络
  - 虚拟线程
标签:
  - Java
  - 网络
  - 原理
---

# Java IO 与 NIO

## 1. 核心概念

三组概念先分清，后面所有模型都是它们的组合：

| 维度 | 取值 | 含义 |
| --- | --- | --- |
| 阻塞 / 非阻塞 | 调用是否等 | 阻塞：数据没到就挂起线程；非阻塞：立刻返回「还没好」 |
| 同步 / 异步 | 谁来搬数据 | 同步：应用自己读缓冲；异步：内核读完通知应用 |
| 流 / 通道 | 数据形态 | 流是单向字节序列；通道双向且可对接 Buffer |

- BIO = 阻塞 + 同步，一连接一线程，线程大部分时间在等数据。
- NIO = 非阻塞 + 同步，一个线程用 Selector 管理成百上千连接，自己轮询就绪事件再搬数据。
- AIO（NIO.2）= 异步 + 通知，理论最省事，但 Linux 上 JDK 实现底层是线程池模拟，生态也基本被 Netty 路线取代，了解即可。

## 2. BIO 基础

- 字节流 `InputStream` / `OutputStream`，字符流 `Reader` / `Writer`（多一层编码转换），套缓冲流 `BufferedInputStream` 减少系统调用——裸读单字节是经典性能坑。`new BufferedInputStream(new FileInputStream(f))` 是装饰器模式的教科书案例，每层只加一种能力。
- 一连接一线程的瓶颈：连接数 = 线程数，线程栈内存与上下文切换先于业务逻辑见顶。适合连接少的内部简单服务，不适合长连接海量在线。

## 3. NIO 三件套

### 3.1 Buffer：一切围绕三个指针

| 状态变量 | 含义 |
| --- | --- |
| capacity | 容量，上限 |
| limit | 当前可操作边界：写模式 = capacity，读模式 = 已写入量 |
| position | 下一个读写位置 |
| mark | 临时标记，`reset()` 回到这里 |

- 核心方法：`flip()` 写完切读（limit 置为当前 position、position 归零），`clear()` 读完切写（position 归零、limit 还原），`compact()` 把未读数据挪到头部再切写，`rewind()` 重读。
- 忘记 `flip()` 直接读是 NIO 第一大新手坑：读到的是 position 之后的旧数据或直接读空。堆外内存 `allocateDirect` 避免一次中转拷贝，但分配回收成本高且不受 GC 直接管理，泄漏排查见第 6 节。

### 3.2 Channel

- `FileChannel`、`SocketChannel`、`ServerSocketChannel`，可配置非阻塞模式；数据永远在 Channel 与 Buffer 之间流转，不直接操作数组。
- `FileChannel` 独有 `transferTo` / `transferFrom`（零拷贝，见第 5 节）与 `map`（内存映射文件）。

### 3.3 Selector：多路复用的入口

- 一个 Selector 注册多个 Channel，关心四种事件：`OP_ACCEPT`、`OP_CONNECT`、`OP_READ`、`OP_WRITE`；`select()` 阻塞到至少一个事件就绪，返回就绪集合逐个处理——单线程（或少数几个线程）扛起大量连接靠的就是它，本质对应操作系统的 epoll/kqueue。
- 可运行的回声服务器骨架：

```java
Selector selector = Selector.open();
ServerSocketChannel server = ServerSocketChannel.open();
server.bind(new InetSocketAddress(8080));
server.configureBlocking(false);
server.register(selector, SelectionKey.OP_ACCEPT);
ByteBuffer buf = ByteBuffer.allocate(1024);
while (true) {
    selector.select();
    Iterator<SelectionKey> it = selector.selectedKeys().iterator();
    while (it.hasNext()) {
        SelectionKey key = it.next();
        it.remove(); // 处理完必须移除，否则重复触发
        if (key.isAcceptable()) {
            SocketChannel ch = server.accept();
            ch.configureBlocking(false);
            ch.register(selector, SelectionKey.OP_READ);
        } else if (key.isReadable()) {
            SocketChannel ch = (SocketChannel) key.channel();
            buf.clear();
            int n = ch.read(buf);
            if (n == -1) { ch.close(); continue; }
            buf.flip();
            ch.write(buf); // 回声：实际项目要处理写不完的情况
        }
    }
}
```

- 骨架的两个留白就是实战重点：`write` 一次写不完要注册 `OP_WRITE` 等可写再续写；读到的字节流**没有消息边界**，粘包拆包必须由应用层协议（长度前缀、分隔符）解决。

## 4. NIO.2 与 AIO

- `Path` / `Files` 取代 `File`：`Files.readAllBytes`、`Files.lines`（流式读大文件）、`Files.walk` 遍历目录，日常文件操作优先用这套。
- `AsynchronousSocketChannel` + `CompletionHandler`：读写完成由回调通知。知道模型即可，生产网络编程直接上 Netty，不要自己造 AIO 轮子。

## 5. 零拷贝

- 传统「读文件 → 经网络发出」要经历磁盘 → 内核缓冲 → 用户缓冲 → 内核 socket 缓冲 → 网卡，多次拷贝与态切换；其中「内核 → 用户缓冲」一趟对只转发不加工的场景纯属浪费。
- `FileChannel.transferTo(position, count, socketChannel)` 解决它：数据在内核内部从文件缓冲直达 socket 缓冲（底层 sendfile），不经过用户态。文件下载、静态资源、消息中间件落盘转发都吃这个红利。

```java
long sent = in.transferTo(0, in.size(), socketChannel); // 单次有上限，大文件要循环累加 position
```

- Netty 的零拷贝是另一个层面（`CompositeByteBuf` 逻辑合并、堆外内存池化），与 sendfile 不是一回事，面试常混为一谈。

## 6. 疑难杂症

- **Selector 空轮询**：老版本 JDK 在 Linux 上有 epoll 已知缺陷，`select()` 无事件也立即返回，CPU 被打满。应对：统计短时间空轮询次数，超阈值就重建 Selector 并把 Channel 迁移过去（Netty 内部就是这么兜底的）。
- **堆外内存泄漏**：`ByteBuffer.allocateDirect` 申请的内存靠 Cleaner 在对象被 GC 后回收，对象活得久内存就还不回去，表现为堆正常但进程 RSS 涨、`OutOfMemoryError: Direct buffer memory`。排查与参数见 [JVM 故障排查](开发语言/Java/JVM/JVM-故障排查.md)：`-XX:MaxDirectMemorySize`、NMT 跟踪。
- **文件关了句柄没关**：Channel/Stream 必须 try-with-resources；Linux 下 `lsof -p <pid>` 看句柄数，「Too many open files」多半是没关流或连接池泄漏，不是系统上限不够。

## 7. 常见误区

- 误区：NIO 一定比 BIO 快。单连接、低并发时两者无区别甚至 BIO 更简单快；NIO 的优势只在「连接多、每连接流量稀疏」时成立。
- 误区：换上 NIO 就自动解决粘包。粘包是 TCP 字节流的本质，任何模型都要自己定协议。
- 误区：AIO 是 NIO 的升级版、更先进就该用。JDK 的 AIO 在主流平台实现与生态都不占优，现实选型是 Netty（NIO 封装）或虚拟线程 + BIO 风格代码。
- 新变化：虚拟线程让「一连接一线程」的阻塞写法的成本大幅下降，内部系统、低并发长连接服务可以直接用同步阻塞代码 + 虚拟线程，省掉 Selector 状态机的复杂度；高并发网关类场景仍是 Netty 的地盘。见 [虚拟线程](开发语言/Java/虚拟线程.md)。

## 小技巧与冷知识

- 【小技巧】`BufferedInputStream` / `BufferedReader` 的默认缓冲是 8192 字节，读大文件时可以显式调大，例如 `new BufferedInputStream(in, 64 * 1024)`，减少系统调用次数。反过来，读大量小文件时没必要盲目调大缓冲，缓冲本身也要占内存。这个默认值是 JDK 文档公开的行为，调参前先确认瓶颈确实在系统调用上。
- 【小技巧】`Files.lines(path)` 返回的是 `Stream<String>`，它底层持有打开的文件句柄，必须用 try-with-resources 包起来，否则文件描述符不会及时释放。同样，`Files.walk`、`Files.list` 返回的流也必须关闭。很多人只记得关 `InputStream`，却忘了 NIO.2 的这些流式 API 同样要关。
- 【小技巧】`SocketChannel` 在非阻塞模式下 `read()` 返回 0 表示“当前没数据可读”，不是连接断开；返回 -1 才是对端关闭。把 0 当成 EOF 处理是 NIO 代码里常见的误判，会导致连接被提前关闭。阻塞模式下 `read()` 才会一直等到有数据或 EOF。
- 【小技巧】`ByteBuffer.slice()` 和 `duplicate()` 生成的新 Buffer 与原 Buffer 共享同一块底层内存，改一边另一边立刻可见。解析协议时可以用 `slice()` 切出消息头、消息体分别处理，不用额外拷贝数组。要注意的是它们各自的 position/limit 是独立的，但数据是同一份。
- 【小技巧】读写二进制协议时先确认字节序：`ByteBuffer` 默认是大端序（`ByteOrder.BIG_ENDIAN`），网络协议通常也是大端，但解析 x86 本地程序写出的二进制文件时要用 `buf.order(ByteOrder.LITTLE_ENDIAN)`。字节序错了，读出的整数会完全变样，而且编译期不会报错，只能靠数据对不上来发现。
- 【小技巧】`FileChannel` 支持散射读和聚集写：`read(ByteBuffer[] dsts)` 一次读满多个 Buffer，`write(ByteBuffer[] srcs)` 一次写出多个 Buffer。做协议解析时可以一个 Buffer 装定长头部、另一个装变长体，一次系统调用完成，比分两次读写少一次往返内核。
- 【小技巧】`Files.copy(Path, OutputStream)`、`Files.copy(InputStream, Path, REPLACE_EXISTING)` 可以直接在文件与流之间搬运，不用自己写 `read/write` 循环。底层对文件到文件的情况会尽量走高效路径，代码也比手写循环短得多，复制文件、保存上传流时优先用它。
- 【冷知识】`ByteBuffer.allocateDirect` 申请的堆外内存不是由 GC 直接回收的，而是靠与之关联的 Cleaner 对象在 Buffer 本身被 GC 后才释放。Buffer 对象如果被长期持有（比如放进静态缓存一直不释放），堆外内存就一直占着，堆内存看起来正常，但进程 RSS 会持续上涨。这就是原文第 6 节堆外泄漏的根源。
- 【冷知识】`ObjectOutputStream` 长时间复用会越写越慢、内存越占越多，因为它内部维护了一张“已写对象”的引用表用于处理循环引用和重复引用，写过的对象都不会被 GC。解决办法是定期调用 `reset()` 清空这张表，代价是后续重复对象会重新完整写出。写日志式序列化流时这个坑很容易踩到。
- 【冷知识】`FileChannel.lock()` 拿到的文件锁在很多操作系统上是建议锁（advisory lock），不是强制锁：它只能拦住同样调用 `lock()` 的程序，拦不住直接读写文件的普通程序。跨进程协调可以用它做互斥，但不能把它当成文件访问的强制屏障。另外，锁是与整个 JVM 关联的，同一 JVM 内两个线程对同一文件加锁不会互相阻塞。

## 8. 面试与自测

- BIO、NIO、AIO 的阻塞/同步属性分别是什么？
- Buffer 的 position/limit/capacity 关系？`flip()` 做了什么？
- Selector 的工作原理？为什么能单线程管万级连接？
- 零拷贝解决的是哪一趟拷贝？`transferTo` 为什么可能要循环调用？
- 粘包的根本原因与三种常用拆包方式？

### 参考答案

1. BIO 是阻塞 + 同步，一连接一线程，线程大部分时间在等数据；NIO 是非阻塞 + 同步，一个线程用 Selector 管理大量连接，自己轮询就绪事件再搬数据；AIO（NIO.2）是异步 + 通知，读写完成由回调通知，但 Linux 上 JDK 实现底层是线程池模拟，生态也被 Netty 路线取代，了解即可。
2. capacity 是容量上限；limit 是当前可操作边界，写模式下等于 capacity，读模式下等于已写入量；position 是下一个读写位置。`flip()` 是写完切读：把 limit 置为当前 position、position 归零，此后才能从头读出刚写入的数据；忘记 `flip()` 直接读是新手第一大坑，会读到旧数据或直接读空。
3. 一个 Selector 注册多个 Channel，关心 OP_ACCEPT、OP_CONNECT、OP_READ、OP_WRITE 四种事件；`select()` 阻塞到至少一个事件就绪，再逐个处理就绪集合，处理完必须移除已处理 key，否则重复触发。它能单线程管万级连接，是因为线程不再为每个连接阻塞等待，而是靠操作系统 epoll/kqueue 多路复用、只在有事件时才处理；但实战仍需处理写不完续写（注册 OP_WRITE）和粘包拆包。
4. 零拷贝省掉的是「内核缓冲 → 用户缓冲」这一趟拷贝：只转发不加工的场景下，数据不必经过用户态。`FileChannel.transferTo()` 让数据在内核内部从文件缓冲直达 socket 缓冲（底层 sendfile），不经过用户态。单次调用有上限，大文件要循环累加 position，直到累计发送量等于文件大小。
5. 根本原因是 TCP 是字节流，没有消息边界，发送端与接收端的分段、合并方式不受应用控制，多个应用消息可能粘在一起或被拆开。常用拆包方式有三种：一是长度前缀，先读定长头部得到消息长度再按长度取体；二是分隔符，用特定字符或字节序列分隔消息；三是定长报文，每条消息固定长度，按长度切分（前两种为正文明确给出，定长为标准补充）。

## 9. 关联知识

- [计算机网络](计算机基础/计算机网络.md)：TCP 字节流语义是粘包、多路复用（epoll）的前置知识。
- [虚拟线程](开发语言/Java/虚拟线程.md)：改变阻塞 IO 的成本模型，影响 NIO 选型。
- [JVM 故障排查](开发语言/Java/JVM/JVM-故障排查.md)：堆外内存与句柄泄漏的排查工具链。
- Netty 是本篇的工程落地：实际项目写网络层直接学 Netty 的 EventLoop、ByteBuf 与编解码器，本篇是其原理底座。
