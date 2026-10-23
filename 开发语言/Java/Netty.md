---
知识点: Netty
层级: 进阶
适用版本: Netty 4.1.x
更新日期: 2026-10-23
关联:
  - Java IO 与 NIO
  - 计算机网络
  - 线程池
标签:
  - Java
  - 网络
  - 原理
---

# Netty

## 1. 核心概念

原生 NIO 能用但难用：Buffer 指针要自己 flip、Selector 空轮询要自己兜底、粘包拆包要自己设计、堆外内存泄漏要自己排查。Netty 是把这套东西工程化封装的标准答案——Dubbo、RocketMQ、gRPC-Java、Elasticsearch 的传输层都是它。[Java IO 与 NIO](开发语言/Java/Java-IO与NIO.md) 是原理底座，本篇只讲 Netty 自己的模型，不重复 NIO 基础。

四个核心抽象，先建立地图：

| 抽象 | 是什么 | 类比 |
|---|---|---|
| EventLoop | 一个线程 + 一个 Selector + 一个任务队列 | NIO 篇里手写的大循环 |
| Channel | 一条连接的抽象 | SocketChannel 的封装 |
| ChannelPipeline | 挂在 Channel 上的处理器链 | 拦截器链 |
| ChannelHandler | 链上的一节处理器 | 一个编解码/业务环节 |

## 2. 线程模型：EventLoop 与无锁化

- `EventLoopGroup` 是线程池：服务端典型配两个，boss 只负责 accept 新连接，worker 负责已建立连接的读写。连接数远小于 EventLoop 数时，一个 EventLoop 同时管成百上千条 Channel。
- 绑定规则是理解 Netty 并发模型的关键：**一条 Channel 从创建到关闭，只由一个 EventLoop 线程处理**。所有 IO 事件和 handler 回调都在这一个线程里串行执行，所以 handler 里处理同一连接的数据天然无并发，不用加锁。
- 代价：**绝不能在 handler 里做阻塞操作**（查数据库、调远程接口、Thread.sleep）。堵住一个 EventLoop，挂在它上面的所有连接一起卡死。耗时业务要扔到业务线程池，或用 `DefaultEventExecutorGroup` 把某个 handler 单独绑到另一组线程上执行。
- 普通任务（execute）和定时任务（schedule）都进 EventLoop 的任务队列，与 IO 事件在同线程穿插执行，所以定时任务里同样不能阻塞。

## 3. Pipeline 与 Handler

- Pipeline 是双向链表：入站事件从 head 往 tail 传，出站操作（write）从 tail 往 head 反着走；顺序由 addLast 的先后决定。
- 事件靠手动接力：入站 handler 处理完要调 `ctx.fireChannelRead(msg)` 才传给下一节，不调就到此为止；异常往 tail 冒泡到 `exceptionCaught`。写数据用 `ctx.writeAndFlush`（从当前节往 head）还是 `channel.writeAndFlush`（从 tail 全程），结果可能不同，排查编解码没生效时先看这个。
- 常用基类：`ChannelInboundHandlerAdapter`（自己管消息释放）、`SimpleChannelInboundHandler`（处理完自动 release，业务 handler 默认用它）、`ByteToMessageDecoder`（攒字节做解码，见第 5 节）。
- 无状态 handler 加 `@ChannelHandler.Sharable` 才能多连接共用一个实例；带成员变量的必须每连接 new 一个，这是新手最常见的并发 bug 来源。

## 4. ByteBuf：与 ByteBuffer 的差别

- **双指针**：readerIndex / writerIndex 分开，不需要 flip，这是对 ByteBuffer 最大的易用性改进。
- **池化**：默认 `PooledByteBufAllocator` 复用堆外内存块，高频分配不打 GC 和系统调用；池化代价是**引用计数**——每节 handler 用完消息要 release，漏放就是内存只涨不降。
- 引用计数规则：谁最后用完谁负责 release；`SimpleChannelInboundHandler` 自动处理入站消息；自己在 handler 里把消息转发给异步线程池时，必须 `retain()` 一份并在用完后 release，否则异步线程读到的是已被回收复用的脏数据。

## 5. 编解码与粘包拆包

TCP 是字节流，没有消息边界（原理见 [计算机网络](计算机基础/计算机网络.md)），Netty 把常见拆包策略做成了现成的帧解码器：

| 解码器 | 协议形态 | 适用 |
|---|---|---|
| `FixedLengthFrameDecoder` | 每帧定长 | 简单定长报文 |
| `DelimiterBasedFrameDecoder` | 分隔符（行）结尾 | 文本协议、按行日志 |
| `LengthFieldBasedFrameDecoder` | 长度域声明后续字节数 | 二进制协议，最常用 |

`LengthFieldBasedFrameDecoder` 的五个参数是最容易配错的地方：`maxFrameLength`（超限直接抛异常防内存打爆）、`lengthFieldOffset`、`lengthFieldLength`、`lengthAdjustment`（长度域的值是否包含头部剩余字节）、`initialBytesToStrip`（解码后砍掉几个头部字节）。配错的表现是错位解码出乱码报文，对着协议文档逐字节数一遍再写。

自定义二进制协议的常规套路：魔数 + 版本 + 长度 + 正文，解码器先校验魔数防非法连接乱发数据，校验不过直接关连接。

## 6. 心跳与空闲检测

长连接必须有心跳，否则中间设备（防火墙、NAT、负载均衡）会悄悄回收空闲连接，表现为「连着但永远收不到消息」。

- `IdleStateHandler(readerIdleTime, writerIdleTime, allIdleTime)` 挂在 pipeline 里，空闲超时触发 `IdleStateEvent` 到 `userEventTriggered`。
- 两种处置策略：服务端常用**读空闲**——超过 N 秒没收到任何数据（含心跳）判定对端已死，直接关连接释放资源；客户端常用**写空闲**——空闲时主动发一个心跳包保活。
- 心跳间隔与判定阈值要错开：如客户端 10 秒发一次，服务端 30 秒无数据才关，留出网络抖动余量；一发一判设成同值会大面积误杀。
- 心跳包要与业务包在协议上可区分（消息类型字段），且不要无意义地全网广播心跳，连接量大时心跳本身就是可观流量。

## 7. 可运行示例：回声服务器

与 NIO 篇手写回声对比，看 Netty 省掉了哪些活（Buffer 管理、Selector 循环、就绪集遍历）：

```java
EventLoopGroup boss = new NioEventLoopGroup(1);
EventLoopGroup worker = new NioEventLoopGroup();
try {
    ServerBootstrap b = new ServerBootstrap()
        .group(boss, worker)
        .channel(NioServerSocketChannel.class)
        .childHandler(new ChannelInitializer<SocketChannel>() {
            @Override protected void initChannel(SocketChannel ch) {
                ch.pipeline()
                  .addLast(new IdleStateHandler(30, 0, 0))          // 读空闲 30 秒
                  .addLast(new LengthFieldBasedFrameDecoder(8192, 0, 4, 0, 4))
                  .addLast(new LengthFieldPrepender(4))            // 出站补长度头
                  .addLast(new EchoHandler());
            }
        });
    b.bind(8080).sync().channel().closeFuture().sync();
} finally {
    boss.shutdownGracefully(); worker.shutdownGracefully();
}

class EchoHandler extends SimpleChannelInboundHandler<ByteBuf> {
    @Override protected void channelRead0(ChannelHandlerContext ctx, ByteBuf msg) {
        ctx.writeAndFlush(msg.retain()); // 回声；retain 后由出站路径释放
    }
    @Override public void userEventTriggered(ChannelHandlerContext ctx, Object evt) {
        if (evt instanceof IdleStateEvent) ctx.close(); // 读空闲判死
    }
}
```

粘包在这个示例里由长度域编解码器解决，handler 拿到的永远是完整一帧。

## 8. 疑难杂症

- **内存只涨不降**：九成是 ByteBuf 引用计数漏放。开启 `-Dio.netty.leakDetection.level=PARANOID`（或 ResourceLeakDetector 的 SIMPLE/ADVANCED 级别）在测试环境定位漏放的分配栈，生产用 SIMPLE 抽样即可，全量 PARANOID 开销大。
- **连接处理莫名卡顿、全员延迟**：某个 handler 里做了阻塞调用堵住 EventLoop。排查思路：看业务线程与 EventLoop 线程的命名（`nioEventLoopGroup-x-y`）和线程栈，确认耗时逻辑跑在哪条线程上。
- **收到半包就业务报错**：拆包器没挂或参数配错，handler 直接按完整报文解析。先打印原始字节核对帧边界，不要先怀疑业务代码。
- **客户端断开服务端不感知**：没配读空闲也没开 TCP keepalive，或 keepalive 的系统默认探测间隔以小时计。长连接一律应用层心跳自己管。

## 9. 常见误区

- 误区：Netty 是异步的，所以 handler 里可以随便阻塞。不行，EventLoop 的串行模型决定了阻塞一节、卡死一片。
- 误区：writeAndFlush 返回了就是对方收到了。它只表示数据进了内核发送缓冲，送达确认要靠应用层 ack。
- 误区：把 Netty 的「零拷贝」理解成 sendfile。它主要是 ByteBuf 层面的逻辑合并与池化，与文件 sendfile 是两回事（见 NIO 篇第 5 节）。

## 10. 面试与自测

- 一个 EventLoop 管理多条 Channel 时，线程安全靠什么保证？
- ChannelPipeline 里入站与出站的传播方向为什么相反？
- LengthFieldBasedFrameDecoder 五个参数分别解决什么问题？
- ByteBuf 引用计数漏放会怎样？怎么检测？
- 心跳应该由哪一端发、哪一端判定？阈值怎么配合？

## 11. 关联知识

- [Java IO 与 NIO](开发语言/Java/Java-IO与NIO.md)：本篇的原理底座，Buffer/Selector/零拷贝都在那里。
- [计算机网络](计算机基础/计算机网络.md)：TCP 字节流与粘包的根源。
- [线程池](开发语言/Java/并发/线程池.md)：业务线程池与 EventLoop 的分工边界。
