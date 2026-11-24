# IO 与网络面试题集（含参考答案）

> 答案按 `prompt/面试回答生成.md` 的六段式生成，选题来自《Java架构师技术栈锚点.md》1.4 IO 与网络（P1）的两个可练单元。岗位默认「高级 Java 后端 / 架构师」，以 JDK 17 / 21、Linux epoll、Netty 4.1 为准。口述版可直接读出声，卡壳处即理解缺口。

## 一、IO 模型

### 1. 【必答】IO 模型：BIO、NIO、AIO 的区别与适用场景？

**口述版**

BIO 是同步阻塞，一个连接一个线程，读不到数据线程就挂住，靠线程数扛并发，连接多了线程爆炸、上下文切换成本高，只适合连接数少且固定的场景。NIO 是同步非阻塞加多路复用，Channel 注册到 Selector，一个线程用 select 轮询多个连接的就绪事件，就绪了才去读写，所以少量线程就能管大量连接，这是现在主流方案，Netty 就是基于它。AIO 是异步非阻塞，发起读写后直接返回，由操作系统完成后回调，理论上最省线程，但 Linux 上 JDK 的 AIO 底层仍靠 epoll 加线程池模拟，收益有限、生态也弱。所以选型上：连接少用 BIO，高并发网络服务用 NIO，AIO 更多是概念和小众场景。

**短版**

BIO 一连接一线程、同步阻塞；NIO 同步非阻塞加 Selector 多路复用，少量线程管大量连接，是主流；AIO 异步回调、由 OS 通知完成，Linux 上实现收益有限；高并发服务选 NIO。

**展开版**

- 机制链：BIO `accept` 后阻塞读 → NIO `register` 到 Selector → `select` 返回就绪集合 → 逐个处理；AIO 提交后由回调通知。
- 关键结构：`SocketChannel`、`Selector`、`SelectionKey`、`AsynchronousSocketChannel`、`CompletionHandler`。
- 触发条件：连接数远超线程可承受量、连接空闲时间长、需要单机支撑上万连接。
- 例外：Linux 的 AIO 由 epoll 加线程池模拟，并非真正的内核异步 IO；Windows 的 IOCP 才是完整实现。
- 对比取舍：BIO 编码简单但扩展性差；NIO 扩展好但编程模型复杂；AIO 编程最简但实现和调试不透明。

**追问预判**

- NIO 是非阻塞还是异步 → 同步非阻塞，读写仍需自己发起，只是不等待。
- AIO 为什么没流行 → Linux 实现是模拟的，性能没有质变，Netty 也没采用。
- 一个线程能管多少连接 → 取决于事件处理耗时，纯转发可以到几万，业务重就要拆线程池。

**常见错误答法**

- 说 NIO 是异步 IO（是同步非阻塞加多路复用）。
- 认为 AIO 在所有平台都是内核级异步（Linux 上是模拟）。

**记忆锚点**

BIO 阻塞一连接一线程 → NIO 多路复用 → AIO 回调 → 主流选 NIO。

---

### 2. 【必答】IO 模型：NIO 的三大组件与 Selector 怎么配合？

**口述版**

NIO 三大组件是 Channel、Buffer、Selector。Channel 是双向通道，可读可写，常见有 SocketChannel、ServerSocketChannel、FileChannel；Buffer 是数据载体，读写共用一块内存，靠 position、limit、capacity 三个指针控制，切换读写要 flip 和 clear；Selector 是多路复用器，把多个 Channel 注册上去，每个注册返回一个 SelectionKey 记录关注的事件，比如 OP_ACCEPT、OP_READ、OP_WRITE。工作流程是：Selector.select 阻塞等待，有事件就返回就绪的 key 集合，遍历 key 判断事件类型再做对应处理。关键点是 Channel 必须配置成非阻塞才能注册到 Selector，处理完要 remove 掉已处理的 key，否则会重复触发。

**短版**

Channel 双向通道、Buffer 用 position/limit/capacity 管理读写、Selector 注册 Channel 并返回 SelectionKey 关注事件；select 返回就绪集合后遍历处理，Channel 必须非阻塞才能注册，处理完要移除 key。

**展开版**

- 机制链：`configureBlocking(false)` → `register(selector, OP_READ)` → `select()` 阻塞 → 返回 `selectedKeys` → 处理并 `remove`。
- 关键结构：`SelectionKey` 的 interestOps 与 readyOps、`ByteBuffer` 的三个指针、`Selector` 底层的 epoll 实例。
- 触发条件：服务端 accept 新连接、客户端可读、写缓冲有空闲。
- 例外：`OP_WRITE` 几乎一直就绪，只在写不出去时才注册，否则会空转烧 CPU。
- 对比取舍：单线程管多连接省资源，代价是任何一个处理阻塞都会拖住所有连接，所以耗时逻辑必须切线程池。

**追问预判**

- 为什么 OP_WRITE 不能一直注册 → 大多数时候可写，会导致 select 一直返回、CPU 空转。
- selectedKeys 为什么要手动 remove → 不移除下一轮还会被认为就绪，造成重复处理。
- Selector 空轮询怎么处理 → Netty 会统计 select 次数，超过阈值重建 Selector 规避 JDK 空轮询 bug。

**常见错误答法**

- 说 Buffer 读写不用切换模式（需要 flip/clear）。
- 在 IO 线程里直接查库或调 RPC（会阻塞整个 Selector）。

**记忆锚点**

Channel + Buffer + Selector → 非阻塞才能注册 → select 拿就绪 key → 处理完 remove。

---

### 3. 【必答】IO 模型：epoll 的水平触发与边缘触发有什么区别？

**口述版**

水平触发是只要缓冲区里还有数据，每次 select 或 epoll_wait 都会通知你，没读完下次还会提醒，编程简单不容易漏事件，代价是可能重复通知。边缘触发是只在状态发生变化时通知一次，从无数据变成有数据才触发，之后不再提醒，所以你必须一次把数据读干净，一直读到返回 EAGAIN 为止，否则剩下的数据可能永远不再通知。ET 的通知次数少、效率略高，但要求非阻塞读加循环读，实现更复杂。Netty 默认用水平触发，因为语义简单可靠；要开边缘触发得显式指定 Epoll 的 ET 模式，并且自己保证读循环正确。选型上除非有明确的性能诉求和成熟实现，业务代码不要轻易上 ET。

**短版**

LT 只要还有数据就持续通知，简单不漏事件但可能重复；ET 只在状态变化时通知一次，必须循环读到 EAGAIN，效率高但实现要求严格；Netty 默认 LT。

**展开版**

- 机制链：内核就绪队列 → LT 每次检查仍就绪就放回 → ET 只在节点状态翻转时入队一次。
- 关键结构：`epoll_ctl` 的 `EPOLLET` 标志、非阻塞 fd、`EAGAIN` 返回值、Netty 的 `EpollChannelOption`。
- 触发条件：高吞吐代理、网关、需要减少系统调用次数的场景。
- 例外：ET 配合阻塞 fd 会卡死在最后一次读上；监听 fd 用 ET 时 accept 也要循环到 EAGAIN。
- 对比取舍：LT 稳定易写，代价是多一次系统调用；ET 减少通知次数，代价是实现错误就会丢事件且难排查。

**追问预判**

- ET 下漏读会怎样 → 数据留在缓冲区但没有新事件，连接看起来"卡住"，直到下次有新数据才可能恢复。
- Netty 为什么默认 LT → 可靠性和可维护性优先，绝大多数场景瓶颈不在通知次数。
- 怎么判断是 LT 还是 ET → 看是否设置 EPOLLET，Netty 里看是否用 EpollEventLoopGroup 与对应 ChannelOption。

**常见错误答法**

- ET 模式下只读一次就返回（会丢数据）。
- 认为 LT 会丢事件（LT 恰恰是不容易漏）。

**记忆锚点**

LT 有数据就通知 → ET 只在变化时通知 → ET 必须读到 EAGAIN → Netty 默认 LT。

---

### 4. 【进阶】IO 模型：select、poll、epoll 的差别，为什么 epoll 高效？

**口述版**

select 有三个问题：fd 集合是位图、上限通常 1024；每次调用都要把整个集合从用户态拷到内核态；返回后还要 O(n) 遍历找出就绪的 fd。poll 用数组加链表替换位图，去掉了数量上限，但拷贝和遍历的问题还在。epoll 做了三件事：一是在内核维护一个事件表，用 epoll_ctl 注册一次，之后不用重复拷贝；二是用红黑树管理监听的 fd，增删查是 O(log n)；三是就绪的 fd 由内核通过回调放进就绪链表，epoll_wait 直接返回这个链表，复杂度只和就绪数量有关，与总连接数无关。所以连接多、活跃比例低时 epoll 优势明显，这也是高并发服务都用它的原因。

**短版**

select 有 1024 上限、每次全量拷贝、O(n) 遍历；poll 去掉上限但拷贝和遍历仍在；epoll 用内核事件表加红黑树加就绪链表，只返回就绪项，复杂度与连接总数无关。

**展开版**

- 机制链：`epoll_create` 建实例 → `epoll_ctl` 增删改监听 → 内核回调把就绪 fd 挂到就绪链表 → `epoll_wait` 取链表。
- 关键结构：红黑树 `rbr`、就绪链表 `rdllist`、`epitem` 回调、水平触发的重新入队逻辑。
- 触发条件：连接数上万、活跃连接占比低、需要频繁增删监听。
- 例外：几乎所有连接都活跃时，epoll 的优势会缩小，甚至不如简单的轮询模型。
- 对比取舍：epoll 换来扩展性，代价是只能用于 Linux，跨平台要抽象出 kqueue、IOCP 等等价实现。

**追问预判**

- epoll 是同步还是异步 → 同步多路复用，仍需应用主动读写，只是通知机制高效。
- 为什么不用 select 了 → fd 上限、重复拷贝和 O(n) 遍历在大连接数下不可接受。
- epoll 的惊群怎么解决 → 用 EPOLLEXCLUSIVE 或 SO_REUSEPORT，Netty 层面靠单线程 accept 规避。

**常见错误答法**

- 说 epoll 是异步 IO（仍是同步多路复用）。
- 认为 epoll 在任何场景都比 select 快（连接少且全活跃时差距不大）。

**记忆锚点**

select 上限 + 拷贝 + 遍历 → poll 去上限 → epoll 事件表 + 红黑树 + 就绪链表。

---

### 5. 【兜底】IO 模型：为什么阻塞 IO 的线程模型撑不住高并发？

**口述版**

阻塞 IO 下每个连接占一个线程，线程在 read 上挂起，什么也做不了。连接数上来后有四个问题：一是线程栈内存，默认一个线程栈一到两兆，一万连接就是十几 G 虚拟内存；二是上下文切换，线程多了 CPU 大量时间花在切换而不是计算；三是线程创建销毁的成本；四是大部分连接是空闲的，线程却在白白占着资源。这就是经典的 C10K 问题。解决办法就是从"一连接一线程"转向事件驱动：用少量线程加多路复用管理大量连接，把阻塞等待变成事件通知。所以 NIO 加 Selector 不是性能优化，而是并发模型的根本变化，代价是业务代码要适应回调和状态机。

**短版**

阻塞 IO 一连接一线程，线程栈内存大、上下文切换重、空闲连接白占资源，这就是 C10K；解法是用多路复用加少量线程做事件驱动，把阻塞等待变成事件通知。

**展开版**

- 机制链：连接建立 → 分配线程 → `read` 阻塞 → 线程休眠 → 连接数上升 → 线程数与内存线性增长。
- 关键结构：线程栈大小 `-Xss`、上下文切换成本、fd 上限 `ulimit -n`。
- 触发条件：长连接、大量空闲连接、单机连接数上万。
- 例外：连接数少或每连接计算量很大时，BIO 加线程池依然简单有效。
- 对比取舍：BIO 编码直观但扩展性差；事件驱动扩展好但把复杂度转移到了业务编排和状态管理。

**追问预判**

- C10K 的本质是什么 → 不是 CPU 不够，而是每连接一线程的资源模型撑不住。
- 线程池能解决吗 → 只能限制总量，连接数超过线程数后新连接只能排队，等于拒绝服务。
- 现在还有 C10M 吗 → 有，需要内核旁路、用户态协议栈等技术，普通业务用 epoll 加多进程就够。

**常见错误答法**

- 认为加机器就能解决（模型问题不是资源问题）。
- 说 BIO 配大线程池就是高并发（线程数受内存和切换成本约束）。

**记忆锚点**

一连接一线程 → 栈内存 + 切换 + 空闲占用 → C10K → 事件驱动。

---

## 二、Reactor 与 Netty

### 6. 【必答】Reactor：单线程、多线程、主从三种模型怎么选？

**口述版**

Reactor 是把 IO 事件分发和处理分离的模式。单线程模型是一个线程既做 accept 又做读写和业务，实现最简单，Redis 就是类似思路，但它扛不住业务耗时，一个慢操作会阻塞所有连接。多线程模型是把业务处理丢给工作线程池，Reactor 线程只负责 IO 事件，这是最常见的形态，Netty 的 Worker 加业务线程池就是这个结构。主从模型是连 accept 都拆出去，主 Reactor 只负责接收连接，把新连接分给多个从 Reactor，每个从 Reactor 管一批连接的读写，Nginx 和 Netty 的 Boss/Worker 就是主从。选型看并发量和业务耗时：连接少用单线程，业务重就用多线程，连接量特别大或 accept 频繁就用主从。

**短版**

单线程一肩挑最简单但怕业务阻塞；多线程把业务丢线程池、Reactor 只管 IO，最常见；主从把 accept 和读写分开，Boss 接收、Worker 处理，适合超高并发；按连接量和业务耗时选。

**展开版**

- 机制链：Acceptor 接收连接 → 注册到 Reactor → Reactor 分发就绪事件 → Handler 处理 → 耗时逻辑切业务池。
- 关键结构：`ServerSocketChannel`、`EventLoopGroup`、`ChannelPipeline`、业务 `ExecutorService`。
- 触发条件：网关、RPC 服务端、IM 长连接、代理转发。
- 例外：单线程模型在纯内存操作且无阻塞时性能很好，不必盲目上多线程。
- 对比取舍：线程越多隔离越好，代价是跨线程传递和状态同步复杂度上升，Netty 靠单连接绑定单线程规避锁。

**追问预判**

- Reactor 和 Proactor 的区别 → Reactor 是就绪通知、应用自己读写；Proactor 是完成通知、由内核完成读写。
- Netty 属于哪种 → 主从 Reactor 多线程模型。
- 业务线程池什么时候必须加 → 有任何可能阻塞或耗时的操作时都必须加。

**常见错误答法**

- 在 Reactor 线程里做数据库操作（会拖住整批连接）。
- 认为线程越多吞吐越高（受下游和切换成本限制）。

**记忆锚点**

单线程全包 → 多线程拆业务 → 主从拆 accept → 耗时必切池。

---

### 7. 【必答】Reactor 与 Netty：EventLoop 与线程模型，一个连接绑定哪个线程？

**口述版**

Netty 的 EventLoopGroup 是一组 EventLoop，每个 EventLoop 内部只有一个线程，负责处理注册在它上面的所有 Channel 的 IO 事件和任务队列。一个 Channel 在注册时会被分配到一个 EventLoop，之后整个生命周期都绑定这个 EventLoop，不会切换，所以同一个连接的所有事件都是串行处理的，业务代码在 Handler 里不需要加锁。一个 EventLoop 可以服务成百上千个 Channel，靠的是多路复用。默认线程数是 CPU 核数乘二，可以通过构造函数指定。要注意的是绑定关系是 Channel 到 EventLoop，不是 Channel 到线程池，所以即使有多个 Worker，单个连接也不会并行处理，想并行只能自己切业务线程池。

**短版**

每个 EventLoop 一个线程，管理多个 Channel；Channel 注册时绑定唯一 EventLoop 且终身不变，所以单连接事件串行、Handler 无需加锁；一个 EventLoop 服务大量连接，靠多路复用。

**展开版**

- 机制链：`ServerBootstrap.group(boss, worker)` → accept 得到 Channel → 从 worker 选一个 EventLoop `register` → 之后所有 IO 与任务都在该线程执行。
- 关键结构：`NioEventLoop` 的 selector 与 taskQueue、`EventLoopGroup` 的 chooser、`SingleThreadEventExecutor`。
- 触发条件：连接建立、读写就绪、用户提交任务、定时任务。
- 例外：`ctx.executor().execute` 提交的任务排在同一 EventLoop 队列里，仍会阻塞后续 IO。
- 对比取舍：单线程串行化免锁、逻辑简单，代价是单个连接的吞吐受单线程限制，需要靠多连接或业务池扩展。

**追问预判**

- 为什么 Handler 不用加锁 → 同一 Channel 的事件永远在同一个 EventLoop 线程执行。
- EventLoop 里能执行阻塞任务吗 → 不能，会阻塞该线程上的所有连接，必须切业务池。
- 线程数设为多少 → 默认核数乘二，IO 密集可适当加大，但仍要压测确认。

**常见错误答法**

- 认为多个 Worker 线程会并行处理同一个连接（绑定单线程）。
- 在 Handler 里用 ThreadLocal 缓存连接状态（跨线程丢失且易泄漏）。

**记忆锚点**

EventLoop = 一线程 → Channel 绑定不变 → 单连接串行无锁 → 阻塞任务切池。

---

### 8. 【必答】Reactor 与 Netty：Pipeline 与 Handler 的顺序，入站出站怎么区分？

**口述版**

Pipeline 是 Channel 上的处理器链，本质上是一条双向链表，Head 和 Tail 是内置节点。入站事件比如读到数据、连接建立、异常，从 Head 往 Tail 方向传播，经过的是 ChannelInboundHandler；出站操作比如 write、flush、close，从 Tail 往 Head 方向传播，经过的是 ChannelOutboundHandler。所以添加顺序决定了执行顺序：入站按添加顺序执行，出站按添加的逆序执行。区分方式是看它实现的是哪个接口，或者看事件方向。常见坑是 ctx.write 和 channel.write 的区别：ctx.write 从当前 Handler 往前找下一个出站处理器，channel.write 从 Tail 开始走完整条链，所以编码器放在前面时两者结果不同。

**短版**

Pipeline 是双向链表，入站从 Head 到 Tail 走 InboundHandler，出站从 Tail 到 Head 走 OutboundHandler；入站按添加顺序、出站按逆序；ctx.write 从当前节点往前，channel.write 从 Tail 开始。

**展开版**

- 机制链：`addLast` 构建链表 → 入站 `fireChannelRead` 向后传递 → 出站 `write` 从当前或 Tail 向前找 Outbound。
- 关键结构：`ChannelPipeline`、`ChannelHandlerContext`、`HeadContext`/`TailContext`、`@Sharable`。
- 触发条件：编解码、心跳、鉴权、日志、异常处理、业务分发。
- 例外：`ctx.write` 不经过当前节点之前的出站处理器，容易漏掉编码；`channel.write` 会走全链。
- 对比取舍：链式设计可插拔、职责清晰，代价是顺序敏感、调试时需要看清事件方向。

**追问预判**

- 编码器该放哪 → 放在业务 Handler 之前（靠近 Head 一侧的入站之后），保证读到的已经是解码后的对象。
- 为什么出站是逆序 → 出站方向与入站相反，从 Tail 向 Head 传播。
- 异常怎么传播 → 入站异常沿 Inbound 向后传，出站异常沿 Outbound 向前传，最终由 Tail 记录未处理异常。

**常见错误答法**

- 认为所有 Handler 都按添加顺序执行（出站是逆序）。
- 混用 ctx.write 和 channel.write 导致编码器被跳过。

**记忆锚点**

入站 Head→Tail → 出站 Tail→Head → 入站顺序、出站逆序 → ctx 局部 / channel 全链。

---

### 9. 【进阶】Reactor 与 Netty：业务线程池怎么切，为什么不能在 IO 线程做耗时操作？

**口述版**

Netty 的 IO 线程负责多路复用和事件分发，一个 EventLoop 上挂着成百上千个连接。如果在 Handler 里做数据库查询、RPC 调用或大文件处理，这个线程就被占住，它管的所有连接都停止响应，表现为大面积超时，而且从线程栈上看只是一次普通调用，很难定位。正确做法是给 Pipeline 加一个 EventExecutorGroup，或者在 Handler 里把任务提交到业务线程池，处理完再通过 ctx.channel().eventLoop().execute 回到 IO 线程写回。切的时候要注意三点：线程池要有界并配拒绝策略；要处理任务顺序，同一连接的多条消息可能乱序；要保证写操作最终回到 EventLoop，避免多线程同时写导致状态竞争。

**短版**

IO 线程一拖就拖住整批连接，所以耗时逻辑必须切业务池：用 EventExecutorGroup 或自建线程池，处理完回到 EventLoop 写回；注意有界队列、消息顺序和写操作线程安全。

**展开版**

- 机制链：IO 线程读到消息 → 提交业务池 → 业务线程处理 → `eventLoop().execute` 回到 IO 线程 → 写回客户端。
- 关键结构：`DefaultEventExecutorGroup`、`ChannelHandlerContext.executor()`、`EventLoop.execute` 的任务队列。
- 触发条件：数据库访问、下游 RPC、加解密、大报文解析、文件读写。
- 例外：纯内存且微秒级的计算不必切池，切池本身有跨线程和队列开销。
- 对比取舍：切池保护 IO 线程，代价是线程切换、顺序问题和背压需要自己处理。

**追问预判**

- 切池后消息会乱序吗 → 会，同一连接的多条消息被不同线程处理，需要按业务键哈希或串行化。
- 怎么保证写回安全 → 所有写操作都通过 `channel.writeAndFlush` 提交回 EventLoop。
- 业务池满了怎么办 → 触发拒绝策略，配合背压停止读取或返回限流错误。

**常见错误答法**

- 在 Handler 里直接同步查库（阻塞 IO 线程）。
- 切池后在业务线程直接操作 Channel 内部状态（非线程安全）。

**记忆锚点**

IO 线程只做事件 → 耗时切业务池 → 处理完回 EventLoop 写 → 注意顺序与有界。

---

### 10. 【兜底】Reactor 与 Netty：BossGroup 和 WorkerGroup 分别做什么，线程数怎么配？

**口述版**

BossGroup 负责接收连接，它上面的 EventLoop 只处理 OP_ACCEPT，接受后把新 Channel 注册到 WorkerGroup。WorkerGroup 负责已建立连接的读写事件和业务 Handler 的执行。这就是主从 Reactor 的落地。线程数上，Boss 通常一个线程就够，因为它只做 accept，除非有多个监听端口或 accept 量极大；Worker 默认是 CPU 核数乘二，IO 密集可以适当调大，但要压测确认，不是越大越好。要注意 Boss 和 Worker 必须是不同的组，如果复用同一个组，accept 和读写会互相影响。另外服务端通常只需要一个 ServerBootstrap，客户端用 Bootstrap 且只有一个组。

**短版**

BossGroup 只处理 accept，WorkerGroup 处理已连接 Channel 的读写；Boss 一般一个线程，Worker 默认核数乘二；两组必须分开，避免 accept 与读写互相干扰。

**展开版**

- 机制链：`ServerBootstrap.group(boss, worker)` → boss 的 EventLoop 注册 ServerSocketChannel → accept 后 `childHandler` 生效 → 新 Channel 注册到 worker。
- 关键结构：`NioEventLoopGroup`、`ServerBootstrap.childGroup/childHandler`、`ChannelInitializer`。
- 触发条件：服务端启动、监听多个端口、需要独立的 accept 处理能力。
- 例外：UDP 服务不需要 accept，Boss 的概念不适用；单端口低连接量时两组也可以合并但不利于隔离。
- 对比取舍：拆分提升隔离性，代价是多一组线程资源，配置不当会浪费。

**追问预判**

- Boss 线程数能设多个吗 → 可以，但只有多端口或 accept 频繁时才有意义。
- childHandler 和 handler 的区别 → `handler` 作用于 ServerChannel，`childHandler` 作用于 accept 出来的子 Channel。
- Worker 线程数调大有什么风险 → 上下文切换和内存上升，且瓶颈常在下游而非 IO。

**常见错误答法**

- Boss 和 Worker 共用一个 EventLoopGroup（accept 与读写互相阻塞）。
- 认为 Worker 线程数越多吞吐越高（受连接活跃度和下游限制）。

**记忆锚点**

Boss 只 accept → Worker 管读写 → 两组分开 → Boss 一个、Worker 核数×2。

---

## 三、Netty 核心机制

### 11. 【必答】Netty 核心机制：ByteBuf 相比 ByteBuffer 的改进，引用计数与泄漏检测怎么做？

**口述版**

ByteBuf 主要有三点改进：一是读写双指针，读用 readerIndex、写用 writerIndex，读写互不干扰，不需要像 ByteBuffer 那样频繁 flip 和 clear；二是支持池化，配合 PooledByteBufAllocator 复用内存，减少分配和 GC，还能用堆外内存避免一次拷贝；三是引用计数，retain 加一、release 减一，归零时归还池或释放内存，这样同一块内存可以在 Pipeline 里零拷贝传递而不必复制。泄漏检测是靠 `-Dio.netty.leakDetection.level` 配置级别，SIMPLE 采样低开销、ADVANCED 采样更细、PARANOID 全量检测，检测到未释放会在日志里打 LEAK 报告和创建时的栈，定位是谁分配了却没释放。

**短版**

ByteBuf 有读写双指针免 flip、支持池化与堆外内存、用引用计数在归零时释放；retain/release 配对，泄漏检测靠 leakDetection.level 的 SIMPLE/ADVANCED/PARANOID 级别输出 LEAK 栈。

**展开版**

- 机制链：`alloc.buffer()` 分配 → `writeXxx` 推进 writerIndex → `readXxx` 推进 readerIndex → `retain` 加计数 → `release` 减计数 → 归零归还池。
- 关键结构：`readerIndex`/`writerIndex`/`capacity`、`AbstractReferenceCountedByteBuf`、`ResourceLeakDetector`。
- 触发条件：编解码、协议拼包、堆外缓冲、跨 Handler 传递。
- 例外：`Unpooled.wrappedBuffer` 包装的数组一般不需要释放；继承 `SimpleChannelInboundHandler` 会自动 release，再手动释放会引用计数下溢。
- 对比取舍：池化加引用计数换来高性能，代价是必须严格配对释放，漏放会泄漏、多放会报错。

**追问预判**

- 什么时候必须手动 release → 把 ByteBuf 传给下一个 Handler 之外自己持有、或在异步任务里使用时。
- 泄漏检测有性能开销吗 → PARANOID 有，生产一般用 SIMPLE 或 ADVANCED。
- 堆外内存 Overflow 怎么排查 → 看 Netty 泄漏报告和 DirectMemory 使用量，常见是没 release 或解码器缓存未清。

**常见错误答法**

- 用 SimpleChannelInboundHandler 又手动 release（会重复释放）。
- 认为 ByteBuf 和 ByteBuffer 一样必须 flip 才能读。

**记忆锚点**

读写双指针 → 池化 + 堆外 → retain/release 归零释放 → 泄漏靠检测级别。

---

### 12. 【进阶】Netty 核心机制：背压怎么处理，写缓冲水位是什么？

**口述版**

背压是消费端处理不过来时让生产端慢下来。Netty 有两个方向：写方向看 `Channel.isWritable`，它由写缓冲水位决定，超过高水位（默认 64KB）就变成不可写，低于低水位（默认 32KB）恢复，配合 `channelWritabilityChanged` 回调做限流；读方向可以关闭 AUTO_READ，自己判断处理能力再手动 `read()`，或者用 `setAutoRead(false)` 加业务队列长度控制。如果不做背压，对端发得快而本地处理慢，消息会堆在 ChannelOutboundBuffer 里，最后 OOM 或者延迟无限增长。实践上就是把"我在忙"这个信号传回上游，比如返回 429、暂停读取、或者丢弃低优先级消息，而不是无限缓冲。

**短版**

写方向用 isWritable 和写缓冲高低水位（默认 64K/32K）触发 writabilityChanged 限流；读方向关 AUTO_READ 按处理能力手动 read；不做背压会堆积在出站缓冲直到 OOM。

**展开版**

- 机制链：写入超过高水位 → `isWritable` 变 false → 回调通知业务停发 → 消费后低于低水位 → 恢复可写。
- 关键结构：`WRITE_BUFFER_WATER_MARK`、`ChannelOutboundBuffer`、`ChannelOption.AUTO_READ`、`IdleStateHandler`。
- 触发条件：下游处理慢、对端发送速率高、批量推送、大报文。
- 例外：设置 `AUTO_READ=false` 后必须自己调用 `read()`，否则永远收不到数据；水位对单个 Channel 生效。
- 对比取舍：背压保护内存和延迟，代价是吞吐下降、需要业务配合实现限流语义。

**追问预判**

- 高低水位为什么不同 → 留出缓冲区间避免在阈值附近频繁抖动。
- 关闭 AUTO_READ 后要注意什么 → 初始化和恢复时必须主动 read，否则连接假死。
- 背压和限流的关系 → 背压是链路级反馈，限流是入口策略，两者配合用。

**常见错误答法**

- 认为 Netty 会自动无限缓冲不出问题（会 OOM）。
- 关掉 AUTO_READ 后忘记恢复读取（连接卡住）。

**记忆锚点**

高水位 64K 不可写 → 低水位 32K 恢复 → writabilityChanged 限流 → 读侧 AUTO_READ 手动控。

---

### 13. 【进阶】Netty 核心机制：FastThreadLocal 与池化分配器解决什么问题？

**口述版**

FastThreadLocal 解决的是 ThreadLocal 在 Netty 场景下的两个问题：普通 ThreadLocal 底层是哈希表，有哈希冲突和探测式清理的开销；Netty 的 FastThreadLocal 给每个线程配一个数组，用全局递增的 index 直接下标寻址，读写是 O(1)，并且在线程销毁时统一清理。代价是必须配合 FastThreadLocalThread 使用，普通线程会自动退化成普通 ThreadLocal 的语义。PooledByteBufAllocator 是内存池，借鉴 jemalloc 的思路把内存分成 arena、chunk、page 多级管理，按规格复用，减少频繁分配和 GC 压力，同时支持堆外内存减少一次拷贝。两者都是为高吞吐网络场景做的优化，普通业务代码没必要照搬。

**短版**

FastThreadLocal 用数组加递增下标做 O(1) 寻址，避免哈希冲突和清理开销，需配合 FastThreadLocalThread；PooledByteBufAllocator 用 arena/chunk/page 分级内存池复用缓冲，降低分配与 GC 开销并支持堆外内存。

**展开版**

- 机制链：FastThreadLocal 分配全局 index → 线程内数组按下标存取 → 线程结束批量清理；池化分配按 size class 从 chunk 切分复用。
- 关键结构：`InternalThreadLocalMap` 的 Object 数组、`PoolArena`、`PoolChunk`、`PoolSubpage`、`SizeClasses`。
- 触发条件：Netty 内部大量使用（EventLoop 状态、缓冲分配）、高并发短缓冲场景。
- 例外：非 FastThreadLocalThread 上 FastThreadLocal 会退化为普通实现，收益消失。
- 对比取舍：极致性能换来实现复杂和内存驻留，池化会抬高常驻内存，小流量服务可关掉池化。

**追问预判**

- FastThreadLocal 会泄漏吗 → 线程存活期间不会自动清理，仍需 remove，只是清理成本更低。
- 池化一定更好吗 → 高分配速率时好，低流量时反而增加常驻内存。
- 怎么关闭池化 → 设置 `ChannelOption.ALLOCATOR` 为 `UnpooledByteBufAllocator`。

**常见错误答法**

- 在普通线程池里用 FastThreadLocal 期待加速（会退化为普通语义）。
- 认为池化内存不需要释放（引用计数归零才归还）。

**记忆锚点**

FastThreadLocal 数组下标 O(1) → 需 FastThreadLocalThread；池化 arena/chunk/page → 降分配降 GC。

---

### 14. 【兜底】Netty 核心机制：TCP 参数怎么配，SO_BACKLOG、TCP_NODELAY、SO_KEEPALIVE 有什么用？

**口述版**

SO_BACKLOG 是 accept 队列长度，三次握手完成的连接如果应用还没来得及 accept，会先放在这个队列里，队列满了新连接会被拒绝或丢弃，高并发短连接场景要调大，默认值往往偏小。TCP_NODELAY 关闭 Nagle 算法，Nagle 会把小包攒起来合并发送，降低网络开销但增加延迟，对延迟敏感的 RPC 和游戏必须打开 TCP_NODELAY，也就是禁用 Nagle。SO_KEEPALIVE 是 TCP 层探活，默认两小时才探测一次，太慢，业务上一般不用它，而是在应用层做心跳，比如 IdleStateHandler 配 Ping/Pong。SO_REUSEADDR 用于快速重启时复用 TIME_WAIT 的端口。

**短版**

SO_BACKLOG 调 accept 队列长度应对突发连接；TCP_NODELAY 关闭 Nagle 降低小包延迟；SO_KEEPALIVE 是 TCP 探活但周期太长，业务用应用层心跳；重启复用端口用 SO_REUSEADDR。

**展开版**

- 机制链：内核完成握手 → 连接进 accept 队列 → 应用 accept 取出 → backlog 满则丢弃或拒绝。
- 关键结构：`ChannelOption.SO_BACKLOG`、`TCP_NODELAY`、`SO_KEEPALIVE`、`SO_REUSEADDR`、`ALLOW_HALF_CLOSURE`。
- 触发条件：高并发接入、低延迟 RPC、移动端弱网长连接、服务快速重启。
- 例外：开了 TCP_NODELAY 会增加小包数量，带宽敏感场景要权衡；keepalive 参数可在内核层调。
- 对比取舍：低延迟与高带宽利用率互相制约，按业务是延迟敏感还是吞吐敏感来定。

**追问预判**

- backlog 满了会怎样 → 新连接被内核丢弃，客户端表现为握手超时或连接重置。
- 为什么 RPC 要关 Nagle → 小包攒批会引入几十毫秒延迟，RPC 请求通常很小。
- 心跳该用 TCP keepalive 吗 → 不建议，周期太长且不可控，应用层心跳更灵活。

**常见错误答法**

- 认为 SO_KEEPALIVE 能快速发现断连（默认两小时）。
- 所有场景都开 TCP_NODELAY（会增加小包和带宽开销）。

**记忆锚点**

BACKLOG 抗突发 → NODELAY 降延迟 → KEEPALIVE 太慢用心跳 → REUSEADDR 快速重启。

---

## 四、零拷贝与内存

### 15. 【必答】零拷贝与内存：什么是零拷贝，Java 里有哪些实现？

**口述版**

零拷贝指减少数据在用户态和内核态之间的复制次数，以及减少 CPU 参与。传统 read 加 write 要四次拷贝、四次上下文切换：磁盘到内核缓冲、内核到用户缓冲、用户到 socket 缓冲、socket 到网卡。零拷贝的目标是让数据不进用户态。Java 里常见三种：FileChannel 的 transferTo 和 transferFrom，底层用 sendfile，数据直接从内核页缓存送到 socket，CPU 拷贝次数降到零；MappedByteBuffer 用 mmap 把文件映射到用户空间，读写像访问内存，但仍有页表映射开销；Netty 的 FileRegion 封装 transferTo，CompositeByteBuf 和 slice 则是在应用层避免不必要的内存复制。要注意零拷贝不等于不拷贝，DMA 拷贝仍然存在。

**短版**

零拷贝减少用户态与内核态之间的数据复制和 CPU 参与；Java 实现有 FileChannel.transferTo（sendfile）、MappedByteBuffer（mmap）、Netty 的 FileRegion 与 CompositeByteBuf/slice；DMA 拷贝仍无法消除。

**展开版**

- 机制链：传统 read/write 四次拷贝 → sendfile 两次（DMA 读 + DMA 写，CPU 不参与）→ 支持 scatter-gather 后可再省一次。
- 关键结构：`FileChannel.transferTo/transferFrom`、`MappedByteBuffer`、`FileRegion`、`CompositeByteBuf`、`slice`。
- 触发条件：文件下发、大报文转发、代理转发、静态资源服务。
- 例外：小文件或需要修改数据时零拷贝收益有限，甚至因为映射开销更慢。
- 对比取舍：省 CPU 和内存带宽，代价是数据不可在用户态修改、实现依赖操作系统能力。

**追问预判**

- transferTo 一定零拷贝吗 → 取决于内核和文件系统，不支持 sendfile 时会退化成普通拷贝。
- mmap 和 sendfile 怎么选 → 需要读改数据用 mmap，纯转发用 sendfile。
- Netty 的零拷贝是同一回事吗 → 不完全，Netty 更多指应用层避免复制（堆外、Composite、slice）。

**常见错误答法**

- 认为零拷贝就是完全不发生任何拷贝（DMA 拷贝仍在）。
- 把 Netty 的 slice 和操作系统零拷贝混为一谈（层次不同）。

**记忆锚点**

四次拷贝 → sendfile 到两次 → mmap 映射 → Netty 应用层免复制。

---

### 16. 【必答】零拷贝与内存：mmap、sendfile、splice 的区别与适用场景？

**口述版**

mmap 是把文件映射到进程的虚拟地址空间，应用读写这块内存就等于读写文件页缓存，省掉了内核到用户的一次拷贝，适合需要随机访问或修改文件内容的场景，比如索引文件、大文件解析，代价是页表建立和缺页中断，文件太大时映射管理也麻烦。sendfile 是内核直接把文件内容送到 socket，数据完全不进用户态，适合纯文件转发，比如静态资源服务、文件下载，但它不支持在传输过程中修改数据，早期版本也不能指定偏移以外的灵活操作。splice 是在两个文件描述符之间通过管道搬数据，也是内核态完成，适合代理转发，能把 socket 到 socket 的路径也变成零拷贝，但要求其中一端是管道。

**短版**

mmap 映射文件到用户空间、省一次拷贝、适合随机读写；sendfile 内核直送 socket、适合纯文件转发但不改数据；splice 通过管道在内核搬数据、适合代理转发。

**展开版**

- 机制链：mmap 建立虚拟映射 → 访问触发缺页 → 读页缓存；sendfile 读页缓存 → DMA 写网卡；splice 通过 pipe buffer 移动页引用。
- 关键结构：`MappedByteBuffer`、`FileChannel.transferTo`、`sendfile` 系统调用、`splice` 的 pipe fd。
- 触发条件：静态文件服务、大文件下载、反向代理、消息中间件的日志转发。
- 例外：sendfile 在需要 TLS 加密时无法直接使用，数据必须进用户态加密，所以 HTTPS 下零拷贝失效。
- 对比取舍：mmap 灵活但有缺页和映射开销；sendfile 极致简单高效但不可改数据；splice 通用性好但依赖管道。

**追问预判**

- HTTPS 为什么用不了 sendfile → 需要用户态加密，数据必须经过应用缓冲区。
- mmap 会导致什么问题 → 缺页中断、文件被截断时访问会触发 SIGBUS、大文件映射占用虚拟地址。
- 代理转发该用哪个 → 纯 TCP 转发用 splice，文件用 sendfile。

**常见错误答法**

- 认为 sendfile 可以边加密边发送（必须走用户态）。
- 认为 mmap 没有拷贝（省的是内核到用户那次）。

**记忆锚点**

mmap 映射可读写 → sendfile 直送不改 → splice 管道搬引用 → HTTPS 破零拷贝。

---

### 17. 【进阶】零拷贝与内存：堆外内存怎么分配和回收，为什么要手动管理？

**口述版**

堆外内存通过 ByteBuffer.allocateDirect 或 Netty 的分配器申请，不在 Java 堆里，不受 GC 直接管理，但持有它的 DirectByteBuffer 对象在堆上。回收靠 Cleaner，对象被回收时触发释放，或者显式调用 Unsafe 的 freeMemory，Netty 则是引用计数归零时归还内存池。手动管理的原因是堆外内存生命周期不受 GC 精确控制，分配和释放都有成本，池化能显著降低开销；另外堆外内存不受 `-Xmx` 限制，只受 `-XX:MaxDirectMemorySize` 约束，一旦泄漏表现为物理内存上涨、进程被 OOM Killer 杀掉，而堆 dump 里看不出来。所以要么用池化加引用计数严格配对，要么用 try-finally 或 Cleaner 保证释放。

**短版**

堆外用 allocateDirect 或 Netty 分配器申请，靠 Cleaner 或引用计数归零释放；不受 Xmx 限制但受 MaxDirectMemorySize 约束；手动管理是因为生命周期不受 GC 精确控制、分配成本高，池化可复用且避免泄漏。

**展开版**

- 机制链：`allocateDirect` 调 `Unsafe.allocateMemory` → 建 DirectByteBuffer → 关联 Cleaner → 对象不可达时 Cleaner 释放。
- 关键结构：`DirectByteBuffer`、`Cleaner`、`MaxDirectMemorySize`、Netty `PooledByteBufAllocator`、`PlatformDependent`。
- 触发条件：网络 IO 缓冲、文件映射、大对象缓存、序列化。
- 例外：Full GC 会间接推动 Cleaner 执行，所以有时"重启前内存不降"其实是没触发 GC。
- 对比取舍：堆外减少一次拷贝、不受堆大小限制，代价是分配慢、泄漏难查、需要手动回收。

**追问预判**

- 堆外内存泄漏怎么查 → 看进程 RSS、`jcmd VM.native_memory`、Netty 泄漏报告，堆 dump 帮助有限。
- 为什么 Netty 不用直接回收 → 池化复用比重分配更快，引用计数能精确控制归还时机。
- 能把堆外内存调很大吗 → 受物理内存和其他进程限制，过大会被 OOM Killer 杀。

**常见错误答法**

- 认为堆外内存由 GC 自动回收、不会泄漏（只是延迟回收）。
- 用 `-Xmx` 限制堆外内存（要用 MaxDirectMemorySize）。

**记忆锚点**

allocateDirect → Cleaner 释放 → 受 MaxDirectMemorySize → 泄漏看 RSS 不看堆。

---

### 18. 【兜底】零拷贝与内存：CompositeByteBuf 与 slice 为什么算零拷贝？

**口述版**

CompositeByteBuf 是把多个 ByteBuf 逻辑上拼成一个视图，底层不复制数据，只是在内部维护一个组件列表和偏移映射，读的时候按偏移定位到对应的组件。所以把协议头和消息体拼包时不用真的 new 一块大内存再 copy 两次，省掉内存和 CPU。slice 是从原缓冲区切一段出来，共享同一块底层内存，只是拥有独立的读写指针，修改会互相影响。这两个都属于应用层零拷贝：不减少系统调用，但避免了用户态内的内存复制。要注意 CompositeByteBuf 的组件仍然要正确 release，slice 的生命周期依赖原缓冲区，原缓冲区被释放后 slice 也不能再用。

**短版**

CompositeByteBuf 用组件列表加偏移做逻辑拼接，不复制数据；slice 切出共享底层内存的视图、指针独立；两者都是应用层零拷贝，省用户态复制，但引用计数和生命周期仍需管理。

**展开版**

- 机制链：`addComponent` 记录组件与偏移 → 读取时按全局索引定位组件 → 只在必要时 consolidate 真正合并。
- 关键结构：`CompositeByteBuf` 的 `components` 数组与 `Component` 偏移、`slice(index, length)`、`retainedSlice`。
- 触发条件：协议头加负载拼包、TLS 记录分片、多段数据聚合发送。
- 例外：调 `consolidate()` 会真正复制合并，之后就失去零拷贝收益。
- 对比取舍：免复制提升性能，代价是索引计算更复杂、引用计数容易出错、调试不直观。

**追问预判**

- 什么时候会退化成复制 → 主动 consolidate、或写入操作需要扩容时。
- slice 和 duplicate 的区别 → slice 是子区间视图，duplicate 是整段共享但读写指针独立。
- 引用计数怎么管 → 用 retainedSlice 增加引用，用完 release，避免原缓冲区提前释放。

**常见错误答法**

- 认为 CompositeByteBuf 一创建就复制了所有数据（是逻辑视图）。
- 释放原缓冲区后继续使用 slice（生命周期依赖原缓冲）。

**记忆锚点**

Composite 组件视图 → slice 共享内存 → 应用层免复制 → 引用计数仍要管。

---

## 五、编解码与粘包拆包

### 19. 【必答】编解码：粘包拆包是怎么产生的，怎么解决？

**口述版**

TCP 是字节流，没有消息边界，应用层写两次不代表对端收两次，内核可能把两次写合并成一个包发出去，这叫粘包；也可能一个大包被拆成多段到达，这叫拆包，还可能是两者混合。根因是 TCP 只看字节流，不看业务消息边界，UDP 有边界但不可靠。解决办法有三种：定长消息，每条固定长度，简单但浪费空间、不适合变长；分隔符，用特殊字符或字符串切分，适合文本协议，但要处理转义和分隔符出现在内容里的情况；长度字段，在消息头里写明长度，先读头再按长度读体，这是最通用也最常用的方案，Netty 的 LengthFieldBasedFrameDecoder 就是做这个的。所以协议设计阶段就要把边界定清楚。

**短版**

TCP 是字节流没有消息边界，写两次可能合并、大包可能被拆；解决靠定长、分隔符或长度字段，最通用的是头部带长度，用 LengthFieldBasedFrameDecoder 处理。

**展开版**

- 机制链：应用 write → 内核发送缓冲按 MSS 分段 → 对端接收缓冲合并 → 应用 read 得到任意长度的字节块。
- 关键结构：`ByteToMessageDecoder` 的累积缓冲、`LengthFieldBasedFrameDecoder`、`DelimiterBasedFrameDecoder`、`FixedLengthFrameDecoder`。
- 触发条件：自定义协议、RPC 报文、IM 消息、文件传输。
- 例外：UDP 数据报本身有边界，不会粘包，但会丢包和乱序。
- 对比取舍：定长实现最简单但不灵活；分隔符适合文本但需转义；长度字段最通用但要多读头部。

**追问预判**

- 为什么本地测试没问题线上才有 → 本地数据量小、发送间隔大，恰好一个包一条消息，网络一忙就暴露。
- 分隔符方案有什么风险 → 内容里出现分隔符会切错，需要转义或改用长度字段。
- Netty 的累积缓冲会不会无限增长 → 会，需要设置最大帧长度防止恶意超大包。

**常见错误答法**

- 认为 TCP 一次 write 对应一次 read（没有这个保证）。
- 只用字符串 split 解析，不考虑半包（会解析失败或丢数据）。

**记忆锚点**

TCP 无边界 → 定长/分隔符/长度字段 → 长度字段最通用 → 设最大帧长。

---

### 20. 【必答】编解码：长度字段解码器怎么设计，参数怎么理解？

**口述版**

LengthFieldBasedFrameDecoder 有四个关键参数：lengthFieldOffset 是长度字段相对于帧起始的偏移；lengthFieldLength 是长度字段占几个字节；lengthAdjustment 是长度字段的值需要额外加减多少才等于整帧剩余长度，用来处理长度字段只表示消息体、不包含头部的情况；initialBytesToStrip 是解码后要丢弃的前多少个字节，通常把长度头剥掉，业务只拿消息体。举个最常见的例子：两字节长度加消息体、长度值只表示体长，那 offset 是 0、length 是 2、adjustment 是 0、strip 是 2。配错的表现是解析错位、帧永远凑不齐或者一次吃掉多帧。另外要设 maxFrameLength 防止恶意超大长度导致内存被打爆。

**短版**

四个参数：长度字段偏移、长度字段字节数、长度修正值（把长度换算成整帧剩余长度）、解码后剥离字节数；再配 maxFrameLength 防超大包；常见两字节头加体、长度只含体时 offset 0、len 2、adj 0、strip 2。

**展开版**

- 机制链：累积字节 → 按 offset 读长度字段 → 加 adjustment 得帧长 → 凑够则切帧 → 剥掉 strip 字节 → 交给下一个 Handler。
- 关键结构：`maxFrameLength`、`lengthFieldOffset`、`lengthFieldLength`、`lengthAdjustment`、`initialBytesToStrip`。
- 触发条件：自定义二进制协议、RPC 私有协议、游戏协议。
- 例外：长度字段包含头部时 adjustment 要减去头部长度，写反了会少读或多读。
- 对比取舍：参数灵活能适配大多数协议，代价是配置复杂、错误时表现为"卡住"而不是明确报错。

**追问预判**

- 帧一直凑不齐是什么原因 → 长度算大了，或 lengthFieldLength 写错，导致等一个永远不来的长度。
- 一次解析出多帧会怎样 → 解码器会自动循环切分，但业务 Handler 要能处理连续多条消息。
- 为什么必须设最大帧长 → 防止对端声明一个超大长度，导致累积缓冲被撑爆。

**常见错误答法**

- 把 lengthAdjustment 当成固定偏移（它是长度修正，可为负）。
- 不设 maxFrameLength，认为协议可信（会被恶意包打爆）。

**记忆锚点**

offset → length → adjustment 修正 → strip 剥离 → maxFrameLength 兜底。

---

### 21. 【进阶】编解码：解码器为什么要无状态，@Sharable 是什么？

**口述版**

Netty 的 Handler 默认不是共享的，每个 Channel 在初始化时会新建一份，因为大多数解码器持有累积缓冲、状态机或计数器，这些状态属于单个连接，多个 Channel 共享会串数据。@Sharable 是一个标注，表示这个 Handler 无状态、可以安全地被多个 Pipeline 共享，加上它之后 Netty 允许同一个实例被重复添加，没加却重复使用会抛异常。所以判断标准很简单：所有字段都是不可变的、或者只读配置，就可以加 @Sharable；只要有任何随连接变化的状态，就必须每个连接一个实例。常见的无状态 Handler 是日志、空闲检测、简单的出站编码器；LengthFieldBasedFrameDecoder 这种持有累积缓冲的绝对不能共享。

**短版**

Handler 默认每连接一份，因为多数解码器持有累积缓冲等连接级状态；@Sharable 表示无状态可被多个 Pipeline 共享，加了才允许复用实例，有状态的一律不能共享。

**展开版**

- 机制链：`ChannelInitializer.initChannel` 时 `addLast(handler)` → Netty 检查 `isSharable` → 非共享实例重复添加则抛异常。
- 关键结构：`@ChannelHandler.Sharable`、`ChannelHandlerAdapter.isSharable`、`ByteToMessageDecoder` 的 `cumulation` 字段。
- 触发条件：多个 Channel 共用一个 Pipeline 配置、需要全局统计或日志、自定义编解码器。
- 例外：`SimpleChannelInboundHandler` 有自动释放逻辑但仍可无状态共享；有实例字段的统计器必须用原子类型。
- 对比取舍：共享省对象创建和内存，代价是并发安全责任转移到开发者，写错就是数据串线。

**追问预判**

- 有状态 Handler 共享会怎样 → 多个连接的缓冲混在一起，解析错乱或内存泄漏。
- 编码器通常能共享吗 → 无状态的转换型编码器可以，持有序列化上下文的要注意线程安全。
- @Sharable 能保证线程安全吗 → 不能，它只是许可复用，安全性要靠实现保证。

**常见错误答法**

- 给持有累积缓冲的解码器加 @Sharable（会串数据）。
- 认为加了 @Sharable 就自动线程安全（只是允许共享）。

**记忆锚点**

默认每连接一份 → 有状态不能共享 → @Sharable 声明无状态 → 只是许可不是保证。

---

## 六、TCP 与 HTTP

### 22. 【必答】TCP 与 HTTP：三次握手与四次挥手，为什么是三次和四次？

**口述版**

三次握手是客户端发 SYN，服务端回 SYN 加 ACK，客户端再回 ACK。三次是为了双向确认收发能力，并且同步双方的初始序列号：两次的话服务端无法确认客户端能收到自己的包，历史上失效的重复 SYN 还可能让服务端建立无用连接。四次挥手是因为 TCP 是全双工，两个方向要分别关闭：主动方发 FIN 表示自己没有数据要发了，对方回 ACK 表示收到，但它可能还有数据要发，等它发完再发自己的 FIN，最后主动方回 ACK。所以是 FIN、ACK、FIN、ACK 四步。主动关闭的一方会进入 TIME_WAIT，等 2MSL，确保最后的 ACK 能到达、并让旧连接的重复报文在网络中消散。

**短版**

三次握手同步双方序列号并确认双向收发能力，两次无法确认服务端到客户端的通道；四次挥手因为全双工，两个方向各自 FIN/ACK；主动关闭方进 TIME_WAIT 等 2MSL。

**展开版**

- 机制链：SYN → SYN+ACK → ACK 建立连接；FIN → ACK → FIN → ACK 关闭连接。
- 关键结构：序列号与确认号、SYN/FIN/ACK 标志位、半关闭状态、2MSL。
- 触发条件：连接建立、正常关闭、异常关闭（RST）。
- 例外：同时打开或同时关闭时状态机走另一条路径；RST 会直接终止，不经过四次挥手。
- 对比取舍：三次是最小可靠次数，再多浪费往返；四次是半关闭的必然结果，合并 ACK 与 FIN 可变成三次。

**追问预判**

- 第三次握手能带数据吗 → 可以，理论上能减少一次往返，但很多实现不带。
- 为什么 TIME_WAIT 是 2MSL → 一个 MSL 让 ACK 到达，一个 MSL 让重发的 FIN 被处理。
- 服务端大量 CLOSE_WAIT 说明什么 → 应用没调 close，通常是代码漏关连接。

**常见错误答法**

- 说三次握手是为了"防止丢包"（核心是同步序列号和确认双向能力）。
- 认为 TIME_WAIT 是服务端的问题（主动关闭方才有）。

**记忆锚点**

三次同步序列号 → 四次因全双工 → TIME_WAIT 等 2MSL → CLOSE_WAIT 是应用没关。

---

### 23. 【必答】TCP 与 HTTP：TIME_WAIT 过多怎么排查和优化？

**口述版**

TIME_WAIT 出现在主动关闭连接的一方，作用是保证最后的 ACK 能重传、让旧报文消散。过多会占端口，短连接高并发下可能耗尽本地端口，报 cannot assign requested address。排查先确认是谁在主动关闭：服务端 TIME_WAIT 多，通常是服务端主动断连或短连接；客户端多则是客户端频繁建连。用 ss 或 netstat 按状态统计，再看连接来源。优化分几层：应用层最有效，改短连接为长连接、用连接池复用；内核层可以开 tcp_tw_reuse 让新连接复用 TIME_WAIT 端口，但只在客户端侧安全，服务端不要开；调大本地端口范围、减少 tcp_fin_timeout。不要用 tcp_tw_recycle，它在 NAT 环境下会出问题且已被移除。

**短版**

TIME_WAIT 由主动关闭方产生，过多会耗尽端口；先用 ss 统计确认哪一端主动关闭；优化优先改长连接和连接池，客户端可开 tcp_tw_reuse、扩端口范围，别用 tcp_tw_recycle。

**展开版**

- 机制链：主动关闭发 FIN → 收 FIN 回 ACK → 进 TIME_WAIT → 等 2MSL → 释放端口。
- 关键结构：`net.ipv4.tcp_tw_reuse`、`tcp_max_tw_buckets`、`ip_local_port_range`、`tcp_fin_timeout`。
- 触发条件：短连接高 QPS、HTTP 服务端主动 keep-alive 超时、代理层频繁建连。
- 例外：`tcp_tw_reuse` 只对主动发起连接的一方生效，服务端开启基本无意义。
- 对比取舍：复用端口缓解压力，代价是可能误用旧连接的报文，需时间戳支持；长连接最根治但要处理保活。

**追问预判**

- 服务端 TIME_WAIT 多正常吗 → 如果是服务端主动关闭，属于正常现象，重点看是否影响端口和资源。
- tcp_tw_recycle 为什么不能用 → 依赖时间戳，NAT 下会丢弃正常包，新内核已移除。
- 除了内核参数还能怎么改 → 应用层用连接池、HTTP keep-alive 调大、避免频繁建连。

**常见错误答法**

- 无脑开 tcp_tw_recycle（NAT 环境会出故障）。
- 认为 TIME_WAIT 是异常状态（是正常的关闭流程）。

**记忆锚点**

主动关闭方才 TIME_WAIT → 2MSL → 先查谁在关 → 长连接优先，reuse 只给客户端。

---

### 24. 【必答】TCP 与 HTTP：HTTP/1.1 与 HTTP/2 的核心差异？

**口述版**

HTTP/1.1 是文本协议，头部是明文键值对，支持持久连接和管道化，但管道化要求响应按序返回，前一个慢就堵住后面所有请求，这就是应用层的队头阻塞，所以浏览器实际很少用，而是靠开多个连接绕过去。HTTP/2 改成二进制分帧，把请求和响应拆成带流 ID 的帧，同一条 TCP 连接上多个流可以交错传输，实现了多路复用，彻底解决了应用层队头阻塞；头部用 HPACK 压缩，减少重复头部的开销；还支持服务端推送和流优先级。但 HTTP/2 仍然跑在 TCP 上，TCP 层的丢包重传依然会阻塞整条连接，所以才有基于 UDP 的 HTTP/3。迁移上 HTTP/2 通常配合 TLS 使用。

**短版**

HTTP/1.1 文本协议、持久连接、管道化有队头阻塞；HTTP/2 二进制分帧加流 ID 实现单连接多路复用、HPACK 头压缩、服务端推送，解决应用层队头阻塞但仍有 TCP 层阻塞。

**展开版**

- 机制链：1.1 一请求一响应串行 → 2.0 拆帧按 stream 交错 → 接收端按 stream 重组。
- 关键结构：帧头（长度、类型、标志、流 ID）、HPACK 静态与动态表、SETTINGS 帧、流优先级。
- 触发条件：高延迟网络、大量小请求、静态资源加载、网关与客户端升级。
- 例外：HTTP/2 明文（h2c）升级流程复杂，实际多用 TLS 上的 ALPN 协商。
- 对比取舍：多路复用提升单连接利用率，代价是单连接故障影响面更大、实现和调试更复杂。

**追问预判**

- HTTP/2 还有队头阻塞吗 → 应用层没有了，TCP 层仍有，丢包会阻塞同连接所有流。
- 为什么 HTTP/2 要二进制 → 解析更快、边界清晰、便于扩展帧类型。
- 服务端推送现在还用吗 → 浏览器支持有限，实践里更多用预加载提示替代。

**常见错误答法**

- 认为 HTTP/2 彻底解决了所有队头阻塞（TCP 层仍在）。
- 说 HTTP/1.1 不支持长连接（支持 keep-alive）。

**记忆锚点**

1.1 文本 + 串行 → 2.0 二进制分帧 + 多路复用 + HPACK → TCP 层阻塞仍在。

---

### 25. 【进阶】TCP 与 HTTP：HTTPS 握手过程与证书校验？

**口述版**

HTTPS 是 HTTP 加 TLS。以 TLS 1.2 为例：客户端发 ClientHello，带支持的版本、加密套件和随机数；服务端回 ServerHello 选定的套件和随机数，再发证书链；客户端校验证书链，检查签名、有效期、域名和吊销状态，然后生成预主密钥用服务端公钥加密发过去；双方用三个随机数算出会话密钥，再各自发 Finished 验证握手完整性。TLS 1.3 把往返压缩到一次，去掉了 RSA 密钥交换，只保留前向安全的 ECDHE。证书校验的关键是信任链：从站点证书逐级验签到受信任的根 CA，任何一环失败都会告警。工程上还要注意证书过期、SNI 多域名、会话复用和 OCSP 装订。

**短版**

TLS 1.2 两次往返：ClientHello/ServerHello 加证书，客户端验链后用公钥加密预主密钥，双方导出会话密钥并 Finished；TLS 1.3 压到一次往返、只保留 ECDHE；证书校验是从站点证书验签到受信根 CA。

**展开版**

- 机制链：协商版本与套件 → 交换随机数 → 验证书 → 密钥交换 → 导出会话密钥 → Finished 校验。
- 关键结构：证书链、CA 签名、SNI、ALPN、OCSP Stapling、会话票据。
- 触发条件：HTTPS 建连、双向认证、证书轮换、多域名部署。
- 例外：TLS 1.3 的 0-RTT 能进一步省往返，但有重放风险，只适合幂等请求。
- 对比取舍：安全换性能，握手增加往返和计算；会话复用和 TLS 1.3 用来抵消这部分开销。

**追问预判**

- 为什么 TLS 1.3 更快 → 一次往返、密钥交换算法简化、支持 0-RTT 恢复。
- 证书链校验失败常见原因 → 中间证书缺失、过期、域名不匹配、根证书不在信任库。
- 双向认证怎么做 → 服务端要求客户端证书，客户端也要配置密钥和信任链。

**常见错误答法**

- 认为 HTTPS 只是把 HTTP 加密（还包含身份认证和完整性校验）。
- 说 TLS 1.3 仍然用 RSA 密钥交换（已移除，用 ECDHE）。

**记忆锚点**

Hello 协商 → 证书验链 → 密钥交换 → Finished；1.3 一次往返，只留 ECDHE。

---

### 26. 【兜底】TCP 与 HTTP：长连接保活与连接池怎么治理？

**口述版**

长连接的核心问题是"连接看起来还在，其实对端已经没了"，比如 NAT 超时、中间设备断连、服务重启。治理分三块：一是应用层心跳，客户端定时发 Ping，服务端回 Pong，配合 IdleStateHandler 检测读写空闲，超过阈值就关闭连接释放资源；二是连接池，复用连接要设置最大连接数、最大空闲数、空闲回收时间和最大生存时间，还要有借出前的有效性检测，避免拿到死连接；三是超时体系，连接超时、读超时、写超时都要设，且要小于上游的超时，形成递减。另外要注意 DNS 缓存和连接与 IP 的绑定，服务端扩缩容后旧连接会打到已下线的节点。

**短版**

长连接靠应用层心跳加 IdleStateHandler 检测空闲，连接池设最大连接/空闲数与空闲回收，借出前做有效性检测，超时要递减配置，并注意 DNS 缓存导致连接打到旧节点。

**展开版**

- 机制链：建连 → 加入池 → 借出校验 → 使用 → 归还 → 空闲超时或心跳失败 → 关闭移除。
- 关键结构：`IdleStateHandler`、连接池的 maxTotal/maxIdle/minIdle/maxLifetime、心跳定时器、`SO_KEEPALIVE`。
- 触发条件：RPC 调用、数据库连接、HTTP 客户端、消息推送。
- 例外：服务端主动 keep-alive 超时时间要大于客户端空闲回收时间，否则会出现"刚借出就被关"。
- 对比取舍：长连接省握手开销，代价是状态维护和故障检测复杂，需要心跳和池化配合。

**追问预判**

- 心跳间隔怎么定 → 小于中间设备的最短空闲超时，通常几十秒，且要可配置。
- 为什么会出现连接池里的死连接 → 对端已关闭但本地未感知，靠借出前检测和心跳剔除。
- 超时为什么必须递减 → 内层先超时才能返回明确原因，否则外层先断会掩盖真实问题。

**常见错误答法**

- 完全依赖 TCP keepalive 做保活（周期太长，默认两小时）。
- 连接池不设最大生存时间（长连接可能一直打到已下线节点）。

**记忆锚点**

应用层心跳 + Idle 检测 → 池化设上限与回收 → 借出前校验 → 超时递减。

---

