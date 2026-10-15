# Java 开发工程师核心知识点（完整版）

## 目录

1. [Java 基础](#一java-基础)
2. [集合框架](#二集合框架)
3. [并发编程](#三并发编程)
4. [JVM](#四jvm)
5. [Spring 生态](#五spring-生态)
6. [数据库](#六数据库)
7. [缓存（Redis）](#七缓存redis)
8. [消息队列](#八消息队列)
9. [搜索引擎（Elasticsearch）](#九搜索引擎elasticsearch)
10. [分布式与微服务](#十分布式与微服务)
11. [网络编程与计算机网络](#十一网络编程与计算机网络)
12. [操作系统与 Linux](#十二操作系统与-linux)
13. [数据结构与算法](#十三数据结构与算法)
14. [设计模式与架构](#十四设计模式与架构)
15. [安全](#十五安全)
16. [工程化与 DevOps](#十六工程化与-devops)
17. [系统设计与项目能力](#十七系统设计与项目能力)
18. [AI 应用开发](#十八ai-应用开发)
19. [建议学习路线](#建议学习路线)

---

## 一、Java 基础

- **语法与类型**：8 种基本类型、包装类与自动装箱/拆箱（Integer 缓存池）、类型转换、位运算
- **String**：不可变性、字符串常量池与 `intern()`、String / StringBuilder / StringBuffer 的区别
- **面向对象**：封装、继承、多态；抽象类与接口；重载与重写；访问修饰符；Java 只有值传递
- **Object 方法**：equals 与 hashCode 的约定、toString、clone（深拷贝与浅拷贝）
- **关键字**：static、final、transient、volatile、instanceof
- **内部类与枚举**：成员内部类、静态内部类、局部内部类、匿名内部类；枚举（含枚举单例）
- **异常体系**：Throwable、Error 与 Exception、受检与非受检异常、try-with-resources、异常处理规范
- **泛型**：类型擦除、通配符与 PECS 原则
- **反射与注解**：Class 对象、元注解、自定义注解；动态代理（JDK 代理、CGLIB）
- **SPI 机制**：ServiceLoader，以及 JDBC、Dubbo、Spring Boot 中的应用
- **IO / NIO**：字节流与字符流、BIO / NIO / AIO、Buffer、Channel、Selector
- **序列化**：Serializable、serialVersionUID；JSON、Hessian、Protobuf
- **数值精度**：BigDecimal 的正确用法、浮点数精度问题
- **新特性**
  - Java 8：Lambda、函数式接口、Stream、Optional、新日期 API
  - Java 9–17：模块化（JPMS）、var、record、sealed 类、文本块、switch 表达式
  - Java 21：虚拟线程、模式匹配、Record Patterns、Sequenced Collections

## 二、集合框架

- **整体结构**：Collection（List / Set / Queue）与 Map 两大体系，Iterator 迭代器
- **List**：ArrayList 扩容机制、ArrayList 与 LinkedList 的对比
- **HashMap**：数组 + 链表 + 红黑树、hash 扰动、扩容与 rehash、负载因子、线程不安全的表现（JDK 7 头插法死循环）
- **其他 Map**：LinkedHashMap（可实现 LRU）、TreeMap、WeakHashMap、ConcurrentHashMap（JDK 7 分段锁，JDK 8 改为 CAS + synchronized）
- **Set**：HashSet、LinkedHashSet、TreeSet 的底层实现
- **Queue**：PriorityQueue（堆）、ArrayDeque、各类 BlockingQueue
- **工具与坑**：Collections、Arrays；`Arrays.asList` 和 `subList` 的坑；不可变集合（如 `List.of`）
- **其他**：fail-fast 与 fail-safe、Comparable 与 Comparator

## 三、并发编程

- **线程基础**：创建方式、生命周期与状态、sleep / wait / notify / join / yield、线程中断（interrupt）、守护线程
- **线程通信**：wait/notify、Condition、生产者-消费者模型
- **JMM 内存模型**：原子性、可见性、有序性；happens-before；volatile 原理（内存屏障、禁止指令重排）
- **synchronized**：Monitor 原理、锁升级（偏向锁在 JDK 15 起默认禁用，之后被废弃）、锁优化（自旋、锁消除、锁粗化）
- **Lock 体系**：AQS 原理、ReentrantLock（公平与非公平）、ReadWriteLock、StampedLock、Condition
- **CAS 与原子类**：Unsafe、ABA 问题、AtomicXxx、LongAdder
- **并发工具**：CountDownLatch、CyclicBarrier、Semaphore、Exchanger、Phaser
- **并发容器**：ConcurrentHashMap、CopyOnWriteArrayList、ConcurrentLinkedQueue、ConcurrentSkipListMap
- **线程池**：ThreadPoolExecutor 7 个参数、执行流程、拒绝策略、参数配置思路、Executors 默认线程池的隐患、ScheduledThreadPoolExecutor
- **ThreadLocal**：原理、内存泄漏、InheritableThreadLocal / TransmittableThreadLocal
- **异步编程**：Future、CompletableFuture、ForkJoinPool
- **新模型**：虚拟线程（Java 21）、结构化并发
- **问题排查**：死锁的四个必要条件、预防方法与排查

## 四、JVM

- **内存区域**：堆、虚拟机栈、本地方法栈、方法区/元空间、程序计数器、直接内存
- **对象**：创建过程、内存布局（对象头、Mark Word）、访问定位
- **class 与字节码**：class 文件结构、常用字节码指令
- **类加载**：加载过程（加载、验证、准备、解析、初始化）、类加载器、双亲委派及打破方式（Tomcat、SPI）
- **垃圾回收基础**：可达性分析、GC Roots、强软弱虚四种引用
- **GC 算法**：标记-清除、复制、标记-整理、分代收集
- **GC 实现细节**：安全点、STW、三色标记、卡表与记忆集
- **垃圾收集器**：Serial、Parallel、CMS（已于 JDK 14 移除）、G1（当前主流）、ZGC（低延迟，JDK 21 起支持分代）
- **执行引擎**：解释执行与 JIT 编译、逃逸分析（栈上分配、标量替换）
- **AOT**：GraalVM Native Image
- **调优与排查**：常用参数、GC 日志分析、OOM / CPU 飙高 / 频繁 Full GC 的排查思路
- **工具**：jps、jstat、jmap、jstack、jcmd、Arthas、MAT、JFR

## 五、Spring 生态

- **Spring Core**：IoC / DI、BeanFactory 与 ApplicationContext、FactoryBean、Bean 生命周期、作用域、循环依赖与三级缓存
- **扩展点**：BeanPostProcessor、BeanFactoryPostProcessor、Aware 接口、事件机制
- **AOP**：原理（动态代理）、切点、通知类型、执行顺序
- **事务**：声明式事务、传播行为、隔离级别、事务失效的常见场景
- **Spring MVC**：DispatcherServlet 请求流程、拦截器与过滤器、全局异常处理、参数校验
- **Spring Boot**：自动配置原理、Starter 机制、配置加载顺序、Actuator 监控
- **Spring Boot 3**：最低要求 JDK 17、javax 迁移到 jakarta
- **其他常用模块**：Spring Data、Spring Security、`@Async` 与 `@Scheduled`、WebFlux 响应式编程
- **Spring Cloud / Alibaba**：Nacos、OpenFeign、Gateway、Sentinel、Seata、LoadBalancer
- **设计模式在 Spring 中的应用**：工厂、单例、代理、模板方法、观察者、适配器

## 六、数据库

- **SQL 基础**：各类 JOIN、子查询、聚合、窗口函数；三范式与表结构设计
- **MySQL 架构**：Server 层与存储引擎、InnoDB 与 MyISAM 的区别
- **索引**：B+ 树、聚簇与非聚簇索引、回表、覆盖索引、最左前缀、索引下推、索引失效场景
- **事务**：ACID、隔离级别、MVCC（ReadView、undo log 版本链）
- **日志**：undo log、redo log、binlog、两阶段提交
- **锁**：表锁、行锁、间隙锁、临键锁、意向锁、死锁排查
- **SQL 优化**：EXPLAIN、慢查询分析、深分页优化
- **架构扩展**：主从复制、读写分离、分库分表（ShardingSphere）、Canal 数据同步
- **连接池**：HikariCP、Druid
- **ORM**：MyBatis（#{} 与 ${}、一级/二级缓存、插件机制、动态 SQL）、JPA
- **其他存储**：MongoDB、分布式数据库（TiDB）的基本了解

## 七、缓存（Redis）

- **数据结构**：String、List、Hash、Set、ZSet 及底层编码（SDS、listpack、quicklist、skiplist）；特殊类型 Bitmap、HyperLogLog、GEO、Stream
- **线程模型**：单线程执行命令、IO 多路复用、6.0 起支持多线程网络 IO
- **持久化**：RDB、AOF、混合持久化
- **过期与淘汰**：过期删除策略、内存淘汰策略
- **高级特性**：Pipeline、Lua 脚本、事务
- **缓存问题**：穿透、击穿、雪崩及解决方案；布隆过滤器；大 Key 与热 Key 的处理
- **一致性**：缓存与数据库双写一致性（旁路缓存、延迟双删）
- **分布式锁**：SETNX + 过期时间、Redisson 看门狗、RedLock
- **高可用**：主从复制、哨兵、Cluster（哈希槽）
- **本地缓存**：Caffeine、多级缓存架构

## 八、消息队列

- **基本作用**：解耦、异步、削峰
- **主流选型**：Kafka、RocketMQ、RabbitMQ 的对比
- **消费模型**：Push 与 Pull、消费者组
- **可靠性**：消息不丢失、重复消费（幂等）、顺序消息、消息积压处理
- **异常处理**：重试机制、死信队列
- **Kafka 专项**：分区、副本、ISR、零拷贝、高吞吐原理
- **RocketMQ 专项**：事务消息、延时消息

## 九、搜索引擎（Elasticsearch）

- **核心原理**：倒排索引、分词器（IK）
- **基本概念**：索引、文档、Mapping、分片与副本
- **查询**：DSL 查询、聚合分析、相关性评分
- **深分页**：from/size、scroll、search_after
- **数据同步**：与 MySQL 同步（Canal、Logstash）
- **集群**：写入与查询流程、近实时搜索

## 十、分布式与微服务

- **理论**：CAP、BASE；Raft / Paxos 的基本思想
- **分布式事务**：2PC、3PC、TCC、Saga、本地消息表、最大努力通知
- **分布式基础**：分布式 ID（雪花算法、号段模式）、分布式锁、分布式 Session、ZooKeeper
- **RPC**：原理、Dubbo、gRPC
- **服务治理**：注册中心、配置中心、负载均衡算法、一致性哈希
- **稳定性**：限流算法（令牌桶、漏桶、滑动窗口）、熔断、降级、超时与重试
- **分布式调度**：XXL-JOB
- **发布策略**：灰度发布、蓝绿发布
- **可观测性**：链路追踪（SkyWalking、OpenTelemetry）、日志（ELK）、监控告警（Prometheus + Grafana）
- **云原生**：Service Mesh（Istio）的基本概念

## 十一、网络编程与计算机网络

- **网络模型**：OSI 与 TCP/IP 分层
- **TCP**：三次握手、四次挥手、TIME_WAIT、流量控制、拥塞控制、粘包与拆包
- **HTTP**：常用方法与状态码、HTTPS（TLS 握手、证书）、HTTP/1.1 → HTTP/2 → HTTP/3
- **其他协议**：DNS、WebSocket
- **会话管理**：Cookie、Session 与 Token（JWT）的区别
- **Netty**：Reactor 线程模型、EventLoop、ByteBuf、Pipeline、编解码器、零拷贝
- **Nginx**：反向代理、负载均衡、动静分离

## 十二、操作系统与 Linux

- **进程与线程**：进程与线程、上下文切换、进程间通信
- **内存**：虚拟内存、页表
- **IO 模型**：select / poll / epoll、零拷贝（mmap、sendfile）
- **Linux 命令**：top、ps、netstat/ss、grep、awk、tail、free、df、iostat
- **排障**：日志分析、性能排查

## 十三、数据结构与算法

- **数据结构**：数组、链表、栈、队列、哈希表、树（二叉树、BST、AVL、红黑树、B+ 树）、堆、图、Trie
- **算法**：排序、二分、双指针、滑动窗口、DFS / BFS、回溯、贪心、动态规划
- **复杂度**：时间与空间复杂度分析
- **刷题**：LeetCode Hot 100、剑指 Offer

## 十四、设计模式与架构

- **设计原则**：SOLID、迪米特法则、合成复用原则
- **常用设计模式**：单例、工厂、建造者、代理、装饰器、适配器、策略、模板方法、观察者、责任链
- **架构思想**：分层架构、六边形 / 整洁架构、高内聚低耦合
- **DDD**：领域、聚合、实体、值对象、限界上下文
- **代码质量**：重构手法、代码坏味道识别

## 十五、安全

- **常见攻击与防护**：SQL 注入、XSS、CSRF、SSRF、越权
- **加密**：对称加密（AES）、非对称加密（RSA）、摘要算法（SHA-256）、数字签名
- **认证授权**：Spring Security、JWT、OAuth2、SSO 单点登录
- **数据安全**：数据脱敏、敏感信息加密存储、接口签名与防重放

## 十六、工程化与 DevOps

- **构建工具**：Maven / Gradle（生命周期、依赖冲突处理、私服）
- **版本管理**：Git 常用命令、分支策略（Git Flow）、冲突解决
- **日志**：SLF4J + Logback / Log4j2、日志规范
- **测试**：单元测试（JUnit 5、Mockito）、集成测试（Testcontainers）、基准测试（JMH）、压测（JMeter）
- **代码质量**：编码规范（阿里巴巴 Java 开发手册）、SonarQube、Code Review
- **接口文档**：Swagger / OpenAPI
- **容器与部署**：Docker、Kubernetes 基础、CI/CD（Jenkins、GitLab CI）

## 十七、系统设计与项目能力

- **经典场景**：秒杀系统、短链接、Feed 流、排行榜、订单超时取消、IM 系统
- **接口设计**：RESTful、幂等、防重、限流、鉴权
- **高并发与高可用**：缓存、异步、拆分、冗余、降级
- **性能**：压测、性能瓶颈定位与优化
- **线上治理**：问题排查、故障复盘、容量评估

## 十八、AI 应用开发

- **框架**：Spring AI、LangChain4j
- **大模型基础**：大模型 API 调用、Prompt 设计、Token 与上下文
- **应用模式**：RAG（检索增强生成）、向量数据库（Milvus、pgvector）、Function Calling、Agent
- **AI 辅助开发**：用 AI 编程助手提升编码、测试和排障效率

---

## 建议学习路线

| 阶段 | 内容 |
| --- | --- |
| 入门 | Java 基础 → 集合 → 并发 → JVM |
| 进阶 | MySQL → Redis → Spring 全家桶 |
| 高级 | MQ / ES → 分布式与微服务 → 系统设计 → AI 应用开发 |
| 贯穿始终 | 计算机网络、操作系统、数据结构与算法 |
