# Netty 核心知识点学习文档

Oct 23, 2026 · @meteorzhong

## 导读

本文展开《知识图谱.html》「Web 与框架」分类中的 Netty 节点，并覆盖 TCP 粘包拆包。标注沿用图谱：**【核心】** 面试必问；**【重点】** 常被追问；**【了解】** 知道概念即可。默认版本为 Netty 4.1 + JDK 17。

| 图谱节点 | 级别 | 本文章节 |
| --- | --- | --- |
| Netty（netty） | 重点 | 线程模型、粘包拆包、ByteBuf 与零拷贝、Pipeline、心跳与调优 |

**复习顺序**：主从 Reactor 线程模型 → EventLoop 与 Pipeline → ByteBuf 与零拷贝 → 粘包拆包 → 编解码与心跳 → 内存池与泄漏排查。面试的主线通常是「为什么不用原生 NIO / 为什么 Netty 快」，能答出线程模型、零拷贝、内存池三件事即可。

## 一、线程模型

Netty 用「主从 Reactor 多线程模型」把连接建立与 I/O 处理分开：`bossGroup` 只负责 Accept，`workerGroup` 负责已建立连接的读写和业务 Handler。

### 1.1 主从 Reactor 模型【核心】

| 角色 | Netty 中的实现 | 职责 |
| --- | --- | --- |
| 主 Reactor | `bossGroup`（`NioEventLoopGroup`） | 监听端口、处理 `OP_ACCEPT`、把新连接注册到 worker |
| 从 Reactor | `workerGroup` | 处理 `OP_READ` / `OP_WRITE`，执行 ChannelPipeline 中的 Handler |
| Acceptor | `ServerBootstrapAcceptor` | boss 收到连接后，选一个 worker EventLoop 并注册 Channel |

相比「单 Reactor 单线程」（一个线程做所有事，任一慢操作都会阻塞）和「单 Reactor 多线程」（业务丢线程池，连接与读写仍在一个 Reactor，高并发下仍是瓶颈），主从模型把 Accept 与读写分离，能充分利用多核。

### 1.2 EventLoop 与 EventLoopGroup【重点】

- **EventLoop** = 一个线程 + 一个 Selector + 一个任务队列，三者一一绑定。它既处理 I/O 事件，也执行提交到它上面的普通任务（`execute`）和定时任务（`schedule`）。
- **EventLoopGroup** 是一组 EventLoop。默认线程数为 **CPU 核数 × 2**，可通过构造参数指定；boss 通常 1 个即可（`new NioEventLoopGroup(1)`）。
- **一个 Channel 从注册到关闭只绑定一个 EventLoop**，因此同一 Channel 的所有事件都在同一个线程中串行执行，业务 Handler 里**不需要加锁**，这是 Netty 高性能与易用性的关键设计。
- 不要在 EventLoop 线程里执行耗时操作（数据库查询、远程调用、大文件读写），否则会阻塞该 EventLoop 上所有 Channel。正确做法是把耗时逻辑丢到独立业务线程池，或用 `EventExecutorGroup` 单独承载 Handler。

**`NioEventLoop.run()` 的主循环**（简化）：`select` 等待事件 → 处理就绪的 I/O 事件 → 执行任务队列中的任务 → 处理定时任务，然后进入下一轮。

### 1.3 ChannelPipeline 与 Handler【重点】

Pipeline 是 Channel 内部的一条**双向链表**，节点是 `ChannelHandlerContext`，每个 Context 包装一个 Handler：

- **入站（Inbound）**：事件从 head 到 tail 传播（`channelRead`、`channelActive`、`exceptionCaught`），处理器继承 `ChannelInboundHandlerAdapter` 或实现 `ChannelInboundHandler`。
- **出站（Outbound）**：事件从 tail 到 head 传播（`write`、`flush`、`connect`）。在 Handler 中调用 `ctx.write()` 会从**当前节点往前**找下一个出站处理器；`ctx.channel().write()` 则从 **tail 开始**整条链重新走。
- **顺序很重要**：编解码器要放在业务 Handler 之前（入站方向先解码），出站编码器要在业务 Handler 之前（出站方向先经过它）。
- `ctx` 与 `channel` 的 API 区别、`ctx.executor()` 是否在当前 EventLoop 上，是排查「事件没触发」类问题的常用手段。

## 二、ByteBuf 与零拷贝

### 2.1 ByteBuf 的结构与分类【重点】

ByteBuf 用 `readerIndex` 和 `writerIndex` 把读写位置分开，读完不需要像 `ByteBuffer` 那样 `flip()`，可读区间是 `[readerIndex, writerIndex)`。

| 分类 | 说明 |
| --- | --- |
| Heap ByteBuf | 分配在 JVM 堆，读写要经过 JVM 堆，适合需要 `array()` 的场景 |
| Direct ByteBuf | 分配在堆外直接内存，避免一次堆到内核的拷贝，网络传输首选 |
| Composite ByteBuf | 把多个 ByteBuf 逻辑组合，**不需要复制**，是零拷贝的典型应用 |
| Pooled / Unpooled | 池化复用内存块（默认 `PooledByteBufAllocator`），减少分配与 GC；非池化每次新建 |

常用方法：`readableBytes()`、`retainedSlice()`、`duplicate()`、`readInt()`、`writeBytes()`、`discardReadBytes()`、`retain()` / `release()`。

### 2.2 零拷贝的四个层次【核心】

Netty 的「零拷贝」是多个概念的集合，面试要能分开说：

1. **CompositeByteBuf**：把 header 和 body 两个 ByteBuf 组合成一个逻辑整体，不发生内存复制。
2. **slice / duplicate**：共享同一块底层内存，只创建新的索引视图。
3. **FileRegion + `transferTo`**：底层走 `sendfile`，文件数据直接从内核页缓存发到网卡，不经过用户态。
4. **Direct ByteBuf**：数据在堆外，写 socket 时少一次「堆内存 → 直接内存」的复制。

注意它与操作系统层面「零拷贝（mmap / sendfile）」不是一回事，Netty 是同时利用了这两种优化。

### 2.3 引用计数与内存泄漏【了解】

ByteBuf 基于**引用计数**：创建时 `refCnt=1`，`retain()` 加 1，`release()` 减 1，减到 0 时归还内存池。入站 Handler 如果消费了数据但没往下传（没调用 `ctx.fireChannelRead`），必须自己 `release`，否则泄漏。

- 排查：启动参数 `-Dio.netty.leakDetection.level=paranoid`（或 `simple` / `advanced`），Netty 会在 GC 时打印泄漏 ByteBuf 的分配堆栈。
- 堆外内存泄漏的表现是进程 RSS 持续上涨、`OutOfDirectMemoryError`，可用 `-XX:MaxDirectMemorySize` 限制上限并监控。
- 工具类 `ReferenceCountUtil.release(msg)`、`SimpleChannelInboundHandler`（自动 release）可以减少手写错误。

## 三、粘包与拆包

### 3.1 为什么会粘包、半包【核心】

TCP 是**字节流**协议，没有消息边界。发送方多次 write 的数据可能被合并成一个 TCP 段（粘包），一条消息也可能被拆成多个段到达（半包）。原因包括：Nagle 算法合并小包、MSS 限制、接收缓冲区大小、应用层发送速度。

UDP 有消息边界不存在该问题，但不可靠、有大小限制，所以主流 RPC / IM 仍用 TCP，由应用层协议解决边界。

### 3.2 三种解决方案【核心】

| 方案 | 解码器 | 适用 |
| --- | --- | --- |
| 固定长度 | `FixedLengthFrameDecoder` | 每条消息长度固定，简单但浪费带宽 |
| 特殊分隔符 | `DelimiterBasedFrameDecoder` | 文本协议，如 `\r\n`、Redis 协议 |
| **长度字段** | `LengthFieldBasedFrameDecoder` | 二进制协议、RPC，最通用 |

`LengthFieldBasedFrameDecoder` 的四个参数要能解释清楚：

```java
// 协议：4 字节长度（不含自身）+ 4 字节魔数 + 业务数据
new LengthFieldBasedFrameDecoder(
        1024 * 1024,  // maxFrameLength 最大帧长，防止恶意超大包
        0,            // lengthFieldOffset 长度字段的偏移
        4,            // lengthFieldLength 长度字段本身占几个字节
        0,            // lengthAdjustment 长度值需要额外加/减的字节数
        4);           // initialBytesToStrip 解码后跳过前几个字节（这里是长度字段）
```

- `lengthAdjustment`：长度字段的值只算业务体、不算长度字段本身时，需要补上；若长度值包含了头部，则填负数。
- `initialBytesToStrip`：通常设为 `lengthFieldOffset + lengthFieldLength`，把长度字段剥掉，业务 Handler 只收到 body。
- 超过 `maxFrameLength` 会抛 `TooLongFrameException`，这是防攻击的必要配置。

**自定义解码器**：继承 `ByteToMessageDecoder`，在 `decode` 中判断 `in.readableBytes()` 是否足够，不够就 return（等下一次数据），足够则构造消息并 `out.add()`，最后 `in.markReaderIndex()` / `resetReaderIndex()` 处理半包。

## 四、编解码、心跳与重连

### 4.1 编解码器【重点】

| 类型 | 作用 | 基类 |
| --- | --- | --- |
| 解码器（入站） | 字节 → 消息对象 | `ByteToMessageDecoder` |
| 编码器（出站） | 消息对象 → 字节 | `MessageToByteEncoder` |
| 双向编解码 | 两者合一 | `ByteToMessageCodec` |
| 消息转换 | 消息 → 另一种消息（如 POJO → Protobuf） | `MessageToMessageEncoder/Decoder` |

常见实现：`ProtobufEncoder/Decoder`、`StringEncoder`、`ObjectEncoder`（Java 序列化，生产不推荐）、JSON 编解码（Jackson / FastJson 适配）。编码器要处理**写半包**：`writeAndFlush` 时由 Netty 保证完整写出，不要在业务里手动循环写。

### 4.2 心跳与空闲检测【重点】

`IdleStateHandler(readerIdleSeconds, writerIdleSeconds, allIdleSeconds)` 放在 Pipeline 最前面，在指定时间内没有读 / 写 / 读写时触发 `IdleStateEvent`，在业务 Handler 的 `userEventTriggered` 中处理：

```java
@Override
public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
    if (evt instanceof IdleStateEvent e) {
        if (e.state() == IdleState.READER_IDLE) {
            ctx.close();                    // 读空闲：对端可能已死，关闭连接
        } else if (e.state() == IdleState.WRITER_IDLE) {
            ctx.writeAndFlush(HEARTBEAT);   // 写空闲：主动发心跳
        }
    } else {
        super.userEventTriggered(ctx, evt);
    }
}
```

- 服务端一般只检测 `READER_IDLE`（长时间收不到心跳就断开）；客户端检测 `WRITER_IDLE` 发心跳，并带重连逻辑。
- 应用层心跳 vs TCP KeepAlive：KeepAlive 默认 2 小时，粒度太粗，长连接场景必须自己做应用层心跳。
- 断线重连用 `ChannelFutureListener` 监听 `close`，配合指数退避重新 `bootstrap.connect()`，避免服务端重启时被瞬间大量重连打垮。

## 五、高性能原因与调优

### 5.1 Netty 为什么快【核心】

1. **主从 Reactor + 单线程串行化**：一个 Channel 绑定一个 EventLoop，无锁化设计，避免同步开销。
2. **I/O 多路复用**：Linux 下用 epoll 边缘触发，一个线程管理大量连接。
3. **池化直接内存**：`PooledByteBufAllocator` 复用内存块，减少分配、GC 和堆到内核的拷贝。
4. **零拷贝**：CompositeByteBuf、slice、FileRegion。
5. **高效数据结构**：`FastThreadLocal`（用数组下标代替哈希冲突）、`MpscQueue`（多生产者单消费者无锁队列）、`IntObjectHashMap`。
6. **无锁任务队列 + 批量处理**：任务攒批执行，减少系统调用。
7. **可调参数**：`SO_BACKLOG`、`SO_REUSEADDR`、`TCP_NODELAY`、`SO_KEEPALIVE`、高低水位线（`WRITE_BUFFER_WATER_MARK`）控制背压。

### 5.2 常见调优参数【了解】

| 参数 | 建议 |
| --- | --- |
| `SO_BACKLOG` | 提升到 1024 以上，应对瞬时连接洪峰 |
| `TCP_NODELAY` | 开启，关闭 Nagle，降低小包延迟（RPC / IM 必开） |
| `SO_KEEPALIVE` | 开启作为 OS 层兜底，应用层心跳仍不可少 |
| 写缓冲水位 | 设置高水位触发 `isWritable=false`，防止慢消费者把内存撑爆 |
| `ioRatio` | 默认 50，I/O 与任务各占一半，I/O 密集可调高 |
| EventLoop 线程数 | 默认 CPU×2；有大量耗时 Handler 时改用独立 `EventExecutorGroup` |

### 5.3 空轮询 bug【了解】

JDK 的 epoll 在极少数情况下 `select()` 会立即返回但没有就绪事件（空轮询），导致 CPU 100%。Netty 的处理：统计连续空轮询次数，超过阈值（默认 512）时**重建 Selector**，把原 Channel 重新注册到新 Selector 上，规避该 JDK bug。

## 六、高频面试题与自测清单

### 6.1 高频面试题

| # | 题目 | 答案要点 | 章节 |
| --- | --- | --- | --- |
| 1 | Netty 的线程模型是什么？ | 主从 Reactor；boss Accept、worker 读写；一个 Channel 绑定一个 EventLoop | 一 |
| 2 | 为什么一个 Channel 不需要加锁？ | 同一 Channel 的事件始终在同一个 EventLoop 线程串行执行 | 一 |
| 3 | EventLoop 里能放耗时任务吗？ | 不能，会阻塞该 EventLoop 上所有连接；丢业务线程池 | 一 |
| 4 | Pipeline 的入站、出站顺序？ | 入站 head→tail，出站 tail→head；编解码器位置 | 一 |
| 5 | Netty 的零拷贝指什么？ | Composite、slice、FileRegion/sendfile、Direct Buffer 四层 | 二 |
| 6 | ByteBuf 和 ByteBuffer 的区别？ | 读写索引分离不用 flip；池化、引用计数、复合缓冲 | 二 |
| 7 | 怎么排查 ByteBuf 泄漏？ | 引用计数、leakDetection、SimpleChannelInboundHandler | 二 |
| 8 | 粘包半包的原因和解决方案？ | TCP 字节流无边界；固定长度、分隔符、长度字段 | 三 |
| 9 | LengthFieldBasedFrameDecoder 的参数含义？ | maxFrameLength、offset、length、adjustment、strip | 三 |
| 10 | 心跳怎么做？ | IdleStateHandler + userEventTriggered；应用层心跳优于 TCP KeepAlive | 四 |
| 11 | Netty 为什么比原生 NIO 好用且快？ | 线程模型、内存池、零拷贝、无锁队列、成熟编解码 | 五 |
| 12 | 什么是空轮询 bug？Netty 怎么解决？ | select 空返回导致 CPU 100%；重建 Selector | 五 |

### 6.2 动手实验

- [ ] 用 Netty 写一个 Echo 服务端 + 客户端，打印每个 Handler 的线程名，验证同一 Channel 线程固定
- [ ] 在业务 Handler 里 sleep 5 秒，观察同一 EventLoop 上其他连接是否被阻塞
- [ ] 用 `LengthFieldBasedFrameDecoder` 实现自定义协议，故意拆包发送验证半包处理
- [ ] 用 `IdleStateHandler` + 客户端重连实现断线自动恢复，观察事件触发
- [ ] 开启 `-Dio.netty.leakDetection.level=paranoid`，故意不 release 一个 ByteBuf，查看泄漏日志
- [ ] 用 `CompositeByteBuf` 与普通 `ByteBuf` 各拼接一次 header+body，观察内存分配差异

### 6.3 自测清单

- [ ] 能画出主从 Reactor 模型中 boss / worker / Channel / EventLoop 的关系
- [ ] 能说清 Pipeline 的入站出站顺序，以及 `ctx.write` 与 `channel.write` 的区别
- [ ] 能分别解释 Netty 零拷贝的四个层次
- [ ] 能根据协议写出 LengthFieldBasedFrameDecoder 的五个参数并说明含义
- [ ] 能说出至少 5 个 Netty 高性能的原因
- [ ] 能结合项目讲一次长连接、心跳或粘包问题的处理
