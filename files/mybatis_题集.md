# MyBatis 与持久层 面试题集（含参考答案）

> 答案按 `prompt/面试回答生成.md` 的六段式生成，选题来自《Java架构师技术栈锚点.md》2.6 持久层框架。默认 MyBatis 3.5.x + Spring Boot 3 + MySQL 8，涉及 Hibernate、多数据源的地方单独标注。口述版可直接读出声，卡壳处即理解缺口。

## 一、执行流程与核心组件

### 1. 【必答】执行流程：一次 MyBatis 查询是怎么走完的？

**口述版**

MyBatis 查询从 Mapper 接口方法调用开始，代理对象把方法解析成 namespace 加方法名的 MappedStatement，再交给 SqlSession。SqlSession 只是门面，真正干活的是 Executor：它先算 CacheKey 查二级缓存和一级缓存，没命中才走数据库。然后结合参数和动态 SQL 标签生成 BoundSql，StatementHandler 从连接池拿连接、创建 PreparedStatement 并预编译，ParameterHandler 按 TypeHandler 把参数设进去，执行完 ResultSetHandler 按 ResultMap 或自动映射把结果集转成对象返回。连接在事务提交或会话关闭时才归还，所以卡壳点通常集中在缓存顺序、动态 SQL 生成和结果映射这三处。

**短版**

Mapper 代理 → MappedStatement → SqlSession → Executor（二级/一级缓存）→ BoundSql → StatementHandler 预编译 → ParameterHandler 设参 → ResultSetHandler 映射。

**展开版**

- 机制链：Mapper 方法 → MapperProxy 找到 MappedStatement → SqlSession → Executor.query → 查缓存 → 生成 BoundSql → StatementHandler.prepare → ParameterHandler.setParameters → JDBC 执行 → ResultSetHandler.handleResultSets。
- 关键结构：Configuration 持有全部 MappedStatement；Executor 负责缓存与事务；StatementHandler 分 Routing、Prepared、Callable 三种，默认 Prepared。
- 触发条件：任何 select / insert / update / delete；`flushCache`、`useCache`、`statementType` 会改变执行路径。
- 例外：`@Select` 注解、XML、Provider 三种方式最终都注册成 MappedStatement；纯 JDBC 或 JdbcTemplate 不经过这条链。
- 对比取舍：把 SQL 与映射交给框架换来可维护性，代价是链路长、缓存与代理带来隐式行为，出问题时要能定位到具体环节。

**追问预判**

- 二级缓存和一级缓存谁先查 → CachingExecutor 先查二级，未命中再进 BaseExecutor 查一级，最后才回表。
- 动态 SQL 在什么阶段生成 → Executor 调 MappedStatement.getBoundSql 时，由 SqlSource 结合参数解析，不是启动时。
- 连接什么时候归还 → Spring 事务由 DataSourceTransactionManager 统一管理，事务结束才归还，不是单条 SQL 结束就还。

**常见错误答法**

- 把 SqlSession 说成线程安全、全应用共享一个（概念错）。
- 认为一级缓存是跨 SqlSession 的（层级混淆）。

**记忆锚点**

代理找 Statement → SqlSession 门面 → Executor 查缓存 → BoundSql → Statement/Parameter/ResultSet 三 Handler。

---

### 2. 【必答】执行流程：Mapper 接口没有实现类，为什么能调用？

**口述版**

靠 JDK 动态代理。启动时 MyBatis 扫描 Mapper 接口，把接口信息注册进 MapperRegistry；调用 `getMapper` 时用 MapperProxyFactory 生成代理对象，方法调用进 MapperProxy.invoke，先用「接口全限定名 + 方法名」拼出 statement id 去 Configuration 里找 MappedStatement，再交给 MapperMethod 处理。MapperMethod 做两件事：按方法类型决定走 select、insert、update 还是 delete，以及用 ParamNameResolver 把参数整理成 MyBatis 需要的命名参数，比如 `@Param`、单个对象、集合。所以接口里不能写方法重载，因为 statement id 会冲突；default 方法则直接执行不走代理逻辑。

**短版**

JDK 动态代理：MapperRegistry 注册接口，getMapper 生成 MapperProxy，方法名拼 statement id 找 MappedStatement，MapperMethod 决定命令类型并解析参数；不支持重载。

**展开版**

- 机制链：扫描接口 → MapperRegistry.addMapper → getMapper → MapperProxyFactory.newInstance → invoke → MapperMethod.execute。
- 关键结构：MapperProxy 持有 SqlSession 与 MapperMethod 缓存；MapperMethod 内含 SqlCommand（命令类型）与 MethodSignature（返回类型、RowBounds、ResultHandler）。
- 触发条件：任何通过接口调用的方式；用 `sqlSession.selectList("namespace.id")` 则绕过代理。
- 例外：default 方法由 MapperProxy 直接反射调用；接口继承的父接口方法也会被代理，但 statement id 仍按实际声明接口算。
- 对比取舍：动态代理让接口与 XML 解耦、IDE 可跳转，代价是方法重载不可用、泛型返回值与多参数需要显式 `@Param`。

**追问预判**

- namespace 和方法名怎么对应 → statement id = 接口全限定名 + "." + 方法名，必须与 XML 的 namespace + id 完全一致。
- 多个参数为什么必须加 @Param → 否则只能用 param1、arg0 这类名字，可读性差且易错。
- 代理对象能直接转成实现类吗 → 不能，JDK 代理只能转成接口类型。

**常见错误答法**

- 说 MyBatis 用 CGLIB 代理 Mapper 接口（机制错，默认 JDK 代理）。
- 认为接口方法可以重载（实际会因 id 冲突启动失败）。

**记忆锚点**

MapperRegistry → MapperProxy（JDK 代理）→ statement id → MapperMethod → 命令类型 + 参数解析。

---

### 3. 【进阶】执行流程：SqlSession 线程安全吗？和 Spring 整合后谁在管生命周期？

**口述版**

DefaultSqlSession 不是线程安全的，它持有 Executor、一级缓存和事务状态，多线程共用会出现缓存串数据、连接混用。所以原生用法是每次操作新建、用完关闭。和 Spring 整合后由 SqlSessionTemplate 接管：它本身线程安全，内部用动态代理包住真正的 SqlSession，每次调用先看当前线程有没有绑定事务，有事务就从 TransactionSynchronizationManager 取同一个 SqlSession 复用，没有就临时新建、执行完立刻关闭。这也是为什么同一个事务里多次查询能命中一级缓存，而脱离事务的两次查询缓存等于不存在。要注意别在事务里自己 openSession，否则拿到另一个连接，看不到未提交数据还可能锁等待。

**短版**

DefaultSqlSession 非线程安全；Spring 用 SqlSessionTemplate 代理，事务内复用同一 SqlSession、事务外每次新建即关，所以一级缓存只在事务内有效；不要手动另开 SqlSession。

**展开版**

- 机制链：SqlSessionTemplate 的代理 → SqlSessionInterceptor → 查 TransactionSynchronizationManager 的 SqlSessionHolder → 有则复用、无则 openSession → finally 关闭或释放。
- 关键结构：SqlSessionHolder 注册为事务同步资源；连接由 DataSourceTransactionManager 通过 DataSourceUtils 绑定到线程。
- 触发条件：`@Transactional` 生效时共享会话；无事务时每次 Mapper 调用都是新会话。
- 例外：多数据源要各自的事务管理器与会话工厂，否则绑定关系会错；异步线程、线程池里事务上下文不会自动传递。
- 对比取舍：模板代理省去手动关闭、保证连接归还，代价是会话边界变隐式，一级缓存行为随事务有无而变化，容易误判。

**追问预判**

- 没有事务时一级缓存为什么没效果 → 每次调用新建并关闭 SqlSession，缓存随会话销毁。
- 手动 openSession 会怎样 → 拿到独立连接，读不到本事务未提交数据，写操作还可能互相阻塞。
- 异步线程里能用同一个事务吗 → 不能，事务绑定在 ThreadLocal 上，必须显式传递或拆成独立事务。

**常见错误答法**

- 说 SqlSession 可以做成单例复用（线程安全问题）。
- 认为 Spring 下 MyBatis 一定走一级缓存（无事务时不存在）。

**记忆锚点**

SqlSession 非线程安全 → Template 代理 → 事务内复用 / 事务外新建即关 → 连接归 Spring 管。

---

### 4. 【必答】执行流程：Simple、Reuse、Batch 三种 Executor 有什么区别？

**口述版**

三种执行器差别在 Statement 的复用与批处理方式。SimpleExecutor 每次查询都新建 PreparedStatement，用完关闭，最直接也最常用，默认就是它。ReuseExecutor 按 SQL 文本缓存 Statement，同一条 SQL 第二次执行直接复用，省掉重复预编译，适合同一会话里反复执行相同 SQL 的场景。BatchExecutor 把写操作攒成 JDBC batch，调用 flushStatements 或提交时一次性发给数据库，减少网络往返，但返回值不准，批处理里拿不到准确的影响行数，自增主键回填也要用 useGeneratedKeys 而不是 selectKey。可以在全局配置 defaultExecutorType 或单次 SqlSession 指定，批量导入这类场景收益明显。

**短版**

Simple 每次新建 Statement；Reuse 按 SQL 复用 Statement；Batch 攒 JDBC batch、flush 时批量提交，行数返回不准、selectKey 不可靠，要用 useGeneratedKeys。

**展开版**

- 机制链：SqlSession 创建时按 executorType 实例化 → BaseExecutor 提供一级缓存与事务 → 子类实现 doQuery / doUpdate。
- 关键结构：SimpleExecutor、ReuseExecutor、BatchExecutor 都继承 BaseExecutor；CachingExecutor 是装饰器，负责二级缓存，与这三种正交。
- 触发条件：批量写入、数据导入、定时任务适合 BATCH；普通在线查询用 SIMPLE 即可。
- 例外：BATCH 下 `selectKey` 会逐条执行，失去批处理意义；部分驱动不支持批量返回主键。
- 对比取舍：BATCH 提升吞吐但增加内存与事务时长，失败时整批回滚、定位困难；REUSE 省预编译但 Statement 缓存占内存。

**追问预判**

- BatchExecutor 的返回值是什么 → 通常是 SUCCESS_NO_INFO（-2），不能用来判断影响行数。
- 怎么在 Spring 下用 BATCH → 配置 default-executor-type: batch，或注入 SqlSessionFactory 后 openSession(ExecutorType.BATCH) 手动 flush。
- 二级缓存和 Executor 类型冲突吗 → 不冲突，CachingExecutor 包在任意 Executor 外层。

**常见错误答法**

- 说 BATCH 能拿到准确影响行数（结论错）。
- 把 ReuseExecutor 说成复用数据库连接（概念混淆）。

**记忆锚点**

Simple 新建 → Reuse 按 SQL 复用 → Batch 攒批 flush → 行数不准、selectKey 失效。

---

### 5. 【兜底】执行流程：MyBatis 四大核心对象与插件责任链是什么关系？

**口述版**

四大核心对象是 Executor、StatementHandler、ParameterHandler、ResultSetHandler，分别负责执行调度与缓存、SQL 预编译与执行、参数设置、结果映射。插件只能拦这四类对象上的特定方法，用 `@Intercepts` 和 `@Signature` 声明拦截点，Configuration 注册后由 InterceptorChain 依次调用 Plugin.wrap 生成 JDK 代理，包在目标对象外层。多个插件形成责任链，后注册的包在最外层、先被调用，所以顺序会直接影响 SQL 改写结果。拦截时拿到的 Invocation 里有 target、method、args，调 proceed 才继续向内传递。要注意代理对象不是原类型，插件里判断类型、取属性都要用 MetaObject 或 Plugin 的工具方法。

**短版**

Executor / StatementHandler / ParameterHandler / ResultSetHandler 四类对象；插件用 @Intercepts 声明、Plugin.wrap 生成代理，后注册在外层先执行，靠 Invocation.proceed 传递。

**展开版**

- 机制链：Configuration.addInterceptor → InterceptorChain.pluginAll → 对四类对象逐个 wrap → 调用时进 InvocationHandler → 匹配 @Signature 才拦截，否则直接放行。
- 关键结构：Plugin 实现 InvocationHandler；MetaObject 用于反射读写属性；SignatureMap 按方法签名精确匹配。
- 触发条件：分页、租户隔离、数据权限、SQL 审计、慢 SQL 统计几乎都通过插件实现。
- 例外：拦截 Executor 时注意 CachingExecutor 在外层；拦截 StatementHandler 要注意 RoutingStatementHandler 与真正的 PreparedStatementHandler 不是同一个。
- 对比取舍：插件无侵入、可组合，代价是调试困难、顺序敏感、改写 SQL 易破坏参数映射。

**追问预判**

- 为什么插件顺序会影响结果 → 后注册的在外层，先改写 SQL，内层再基于改写后的语句处理。
- 能拦截 Mapper 接口吗 → 不能直接拦，只能拦四大对象；要拦 Mapper 得自己加 AOP。
- 插件里怎么拿到原始 SQL → 从 Invocation 参数里取 MappedStatement 和 BoundSql，BoundSql.getSql()。

**常见错误答法**

- 认为插件可以拦截任意类任意方法（范围错）。
- 说先注册的插件先执行（顺序反了，先注册在内层）。

**记忆锚点**

四对象 → InterceptorChain 倒序包裹 → 后注册先执行 → Invocation.proceed 放行。

---

## 二、缓存机制

### 6. 【必答】缓存：一级缓存的作用域、命中条件和失效时机？

**口述版**

一级缓存是 SqlSession 级别的，存在 BaseExecutor 的 localCache 里，默认开启、作用域 SESSION。命中条件是同一个 SqlSession、同一条 SQL、同样的参数、相同的 RowBounds 和 Environment，CacheKey 由这些拼出来，注意它不含 statement id，所以同一会话里两个不同 Mapper 写了完全相同的 SQL 也能命中。失效主要有四种：同一会话执行了 insert、update、delete，会清空整个 localCache；commit 或 rollback；手动 clearCache；以及把 localCacheScope 配成 STATEMENT，每次查询结束就清。所以它在 Spring 无事务场景下基本没意义，因为有事务才复用同一个会话。

**短版**

SqlSession 级，BaseExecutor.localCache，默认 SESSION；key = SQL + 参数 + RowBounds + Environment；增删改、commit/rollback、clearCache、STATEMENT 作用域都会失效。

**展开版**

- 机制链：query → 生成 CacheKey → localCache 命中直接返回 → 未命中查库 → 结果放入 localCache。
- 关键结构：PerpetualCache 本质是 HashMap；CacheKey 的乘数哈希把 SQL、参数、RowBounds、Environment 组合起来。
- 触发条件：同一 SqlSession 内重复执行相同查询；Spring 事务内多次调用同一查询。
- 例外：`flushCache=true` 的查询会先清缓存再查；嵌套查询、延迟加载共用同一会话，也会共享缓存。
- 对比取舍：省掉重复查询，但脏读风险随会话变长而上升，长事务里同一条查询可能一直读到旧快照。

**追问预判**

- 为什么更新会清空整个缓存而不是按 key 清 → 无法推断 SQL 影响范围，只能全清，这也是它粒度粗的原因。
- 不含 statement id 会有什么后果 → 不同 Mapper 的相同 SQL 会共享缓存，可能拿到不属于当前语义的结果。
- 怎么关闭 → local-cache-scope: statement，或每次查询 flushCache。

**常见错误答法**

- 说一级缓存跨 SqlSession 共享（作用域错）。
- 认为只有 update 会清缓存，漏掉 commit、rollback、clearCache（失效条件不全）。

**记忆锚点**

SqlSession 级 → key 四要素 → 写操作全清 → 事务内才有效。

---

### 7. 【必答】缓存：二级缓存怎么开、作用域在哪、什么时候失效？

**口述版**

二级缓存作用域是 namespace，也就是一个 Mapper，跨 SqlSession 共享。开启方式是在 XML 里加 `<cache/>`，或用 `@CacheNamespace`，还要在全局配置里允许 cacheEnabled，默认是开的。默认实现是 PerpetualCache 加 LRU 淘汰，size 1024，readOnly 为 false 时会用 SerializedCache 存副本，所以实体建议实现 Serializable。写入不是实时的，走 TransactionalCache，只有事务 commit 才真正刷进缓存，rollback 就丢弃。失效是同一 namespace 的写操作清空该 namespace，跨 namespace 要靠 cache-ref 关联，否则改 A 表不会清 B 的缓存，这也是脏数据的主要来源。

**短版**

namespace 级、跨会话；`<cache/>` 开启，默认 LRU、size 1024、readOnly=false 需可序列化；commit 后才写入，同 namespace 写操作清空，跨 namespace 需 cache-ref。

**展开版**

- 机制链：CachingExecutor.query → TransactionalCacheManager 取对应 namespace 缓存 → 命中返回 → 未命中查库 → 结果暂存 → commit 时 flush 到实际 Cache。
- 关键结构：CachingExecutor（装饰器）、TransactionalCache（事务缓冲）、PerpetualCache（存储）、LRU / FIFO / Soft / Weak 淘汰策略。
- 触发条件：读多写少、单机或数据变化不频繁的字典类表；配合 `useCache`、`flushCache` 精细控制。
- 例外：`readOnly=true` 直接返回共享引用，性能好但调用方修改对象会污染缓存；多实例部署时各节点缓存互不可见。
- 对比取舍：减少数据库压力，代价是一致性弱、粒度是整个 namespace，写完就全清，命中率未必高。

**追问预判**

- 为什么 commit 之后才写缓存 → 事务未提交时数据可能回滚，提前写入会产生脏缓存。
- 多表关联怎么办 → 用 cache-ref 让相关 Mapper 共享同一缓存，或干脆不用二级缓存。
- 集群下能用吗 → 各节点本地缓存，无法互相失效，生产一般改用 Redis 做集中缓存。

**常见错误答法**

- 说二级缓存默认开启、加个注解就生效（默认需要显式声明 cache）。
- 认为写操作只清受影响的那部分缓存（实际清整个 namespace）。

**记忆锚点**

namespace 级 → `<cache/>` 开启 → commit 落盘 → 同 namespace 全清 → 跨 namespace 要 cache-ref。

---

### 8. 【进阶】缓存：生产上为什么很少开二级缓存？

**口述版**

核心是一致性成本和收益不对等。二级缓存是进程内本地缓存，多实例部署时各节点各自一份，A 节点更新只会清自己的，B 节点还返回旧数据，除非引入广播或集中缓存。其次是粒度问题，写操作清空整个 namespace，热点表写多读少时命中率很低，还白白占内存。第三是关联查询，跨表更新不会自动清对方 namespace 的缓存，容易出脏数据。再加上它和 Redis 这类集中式缓存的职责重叠，而 Redis 能做精确 key 失效、跨实例一致、可观测，工程上更可控。所以除非是单机、只读字典表，一般直接关掉二级缓存，缓存交给 Redis 或本地 Caffeine 显式管理。

**短版**

多实例本地缓存无法互相失效、namespace 级全清导致命中率低、跨表更新不联动、与 Redis 职责重叠且难观测；只在单机只读字典场景考虑。

**展开版**

- 机制链：写操作 → 清本节点 namespace → 其他节点无感知 → 读到旧值 → 业务出现不一致。
- 关键结构：本地 PerpetualCache 无跨节点通信能力；cache-ref 只解决同一 JVM 内多 Mapper 共享。
- 触发条件：多副本部署、读写分离、跨表关联查询、写多读少。
- 例外：单机只读配置表、枚举字典，开启后收益明确且风险低，可配合短 flushInterval。
- 对比取舍：本地缓存零网络开销但一致性差；Redis 有网络与序列化成本，但能集中失效、支持大集群。

**追问预判**

- 一定要用怎么降低风险 → 只读表、readOnly、短 flushInterval、单机或按租户分片、上线前压测命中率。
- 和 Redis 缓存怎么分工 → 二级缓存管进程内极热小字典，Redis 管跨实例共享数据，避免同一份数据两处缓存。
- 怎么观测命中率 → Cache 的 getHitRatio、getMissRatio，或自定义统计插件。

**常见错误答法**

- 只答"会脏读"不说多实例与粒度问题（层级缺失）。
- 认为开了二级缓存就一定提升性能（忽略命中率与内存成本）。

**记忆锚点**

本地缓存不跨节点 → 写全清 namespace → 跨表不联动 → 与 Redis 重叠 → 生产默认关。

---

### 9. 【进阶】缓存：事务里一级缓存读到旧数据，怎么定位和规避？

**口述版**

典型场景是同一个 Spring 事务内，先查了一次订单，随后另一个连接或另一个服务把订单改了并提交，本事务再查一次，结果还是旧值，因为第二次走了 SqlSession 的一级缓存。也可能是同事务里混用了 JdbcTemplate 或另一个 SqlSession 更新，MyBatis 这边的缓存没被清。定位方法是打开 MyBatis 日志确认第二次查询没有真正发 SQL，再用连接 id 或 general_log 确认走的是同一会话。规避手段有几种：把 local-cache-scope 改成 STATEMENT，让每次查询后清缓存；在关键读之前调 clearCache；统一持久层入口，别在一个事务里混两套访问方式；对一致性要求高的读直接加 flushCache 或走独立只读事务。

**短版**

同事务同 SqlSession 命中一级缓存，看不到其他连接的提交；看日志确认没发 SQL；用 STATEMENT 作用域、clearCache、统一持久层入口、关键读 flushCache 规避。

**展开版**

- 机制链：事务开始绑定 SqlSession → 第一次查询写入 localCache → 外部提交新数据 → 第二次查询命中缓存 → 返回旧值。
- 关键结构：SqlSessionHolder 与 TransactionSynchronizationManager 的线程绑定；localCache 生命周期等于会话。
- 触发条件：长事务、事务内多次读同一数据、读写混用不同框架、批量任务中边改边查。
- 例外：READ_COMMITTED 下其他事务提交后，新连接能读到；但本会话的缓存不会自动失效。
- 对比取舍：STATEMENT 作用域一致性更好但失去会话内缓存收益；REPEATABLE_READ 下读到旧值反而是语义正确的，要先确认隔离级别预期。

**追问预判**

- 怎么快速确认是缓存问题 → 开 DEBUG 日志看第二次查询有没有 Preparing，或临时 clearCache 对比结果。
- 关闭一级缓存影响大吗 → 只影响同会话重复查询的优化，多数在线接口收益有限，可按需关闭。
- 事务隔离级别和这个有关系吗 → 有，RR 下本来就要求可重复读，RC 下才是"读到旧值"的问题。

**常见错误答法**

- 直接归因于数据库隔离级别，忽略一级缓存（定位错）。
- 用 System.gc 或重启应用解决（答非所问）。

**记忆锚点**

同会话命中 → 外部提交不可见 → 日志确认没发 SQL → STATEMENT / clearCache / 统一入口。

---

## 三、SQL 与安全

### 10. 【必答】SQL 安全：`#{}` 与 `${}` 的区别，预编译为什么能防注入？

**口述版**

`#{}` 会被编译成 PreparedStatement 的占位符问号，参数通过 ParameterHandler 和 TypeHandler 单独设进去，SQL 结构与数据分离，所以即使用户输入带引号和 or 1=1，也只会被当成普通字符串值，不会改变语义，这就是防注入的根本。`${}` 是字符串直接拼接，拼完再交给数据库解析，输入里只要有引号、分号、注释符就能改写 SQL。`${}` 只在不能参数化的位置才用，比如表名、列名、order by 的字段和方向，这些位置 PreparedStatement 不支持占位符。生产规范是默认只用 `#{}`，必须用 `${}` 的地方做白名单校验，模糊查询用 concat 拼值而不是把百分号拼进 SQL。

**短版**

`#{}` 走预编译占位符、参数与 SQL 分离，天然防注入；`${}` 直接拼接可被改写 SQL；只在表名列名排序等标识符位置用，且必须白名单。

**展开版**

- 机制链：`#{}` → SqlSource 生成 `?` → ParameterMapping → ParameterHandler.setString 等 → 数据库按值处理；`${}` → 解析时字符串替换 → 拼进最终 SQL。
- 关键结构：ParameterMapping、TypeHandler、BoundSql；`${}` 的值也来自 additionalParameters，但已经变成 SQL 文本。
- 触发条件：所有接收外部输入的位置都要区分；尤其 like、in、order by、动态表名。
- 例外：`${}` 配 `bind` 或 `<foreach>` 仍属拼接；`@Select` 注解里同样存在。
- 对比取舍：`#{}` 安全但只能用于值位置；`${}` 灵活但把安全责任交给开发者，必须靠规范和校验兜底。

**追问预判**

- like 查询怎么写才安全 → `like concat('%', #{kw}, '%')`，不要写 `'%${kw}%'`。
- `#{}` 能不能用在表名上 → 不能，预编译占位符只支持值，不接受标识符。
- 预编译一定安全吗 → SQL 本身动态拼接的部分仍然危险，防注入看的是数据是否进入 SQL 文本。

**常见错误答法**

- 说 `#{}` 会做转义所以安全（本质是参数化，不是转义）。
- 认为 `${}` 只在 like 里危险，忽略 order by、表名（范围遗漏）。

**记忆锚点**

`#{}` 占位符 → 值与结构分离 → `${}` 拼接 → 只给标识符 + 白名单。

---

### 11. 【必答】SQL 安全：动态 SQL 常用标签和容易踩的坑？

**口述版**

常用的是 if、choose-when-otherwise、where、set、trim、foreach、bind，以及 sql 加 include 做片段复用。if 用来按条件拼 where 子句，where 标签会自动去掉开头多余的 and 或 or，set 会去掉末尾多余的逗号，trim 可以自定义前后缀。foreach 用来拼 in 列表或批量插入，collection 要区分数组写 array、集合写 list、加了 `@Param` 就写参数名。坑主要有几个：字符串判断要写 `name != null and name != ''`；OGNL 里单引号字符和字符串比较容易出错，建议用 `'Y'.equals(status)`；SQL 里的小于号要转义或用 CDATA；include 引用要保证 refid 唯一。

**短版**

if / choose / where / set / trim / foreach / bind / sql+include；where 去多余 and、set 去逗号；foreach 的 collection 按类型取名；注意空串判断、OGNL 比较、`<` 转义。

**展开版**

- 机制链：XML 解析成 SqlNode 树 → apply 时按参数求值 → 拼出最终 SQL 文本 → BoundSql 携带 additionalParameters。
- 关键结构：DynamicSqlSource / RawSqlSource、MixedSqlNode、IfSqlNode、ForEachSqlNode、VarDeclSqlNode（bind）。
- 触发条件：多条件查询、批量操作、可选排序、权限过滤。
- 例外：`<where>` 内部若整体为空则不输出 where；`<foreach>` 的 index 在 list 里是下标、在 map 里是 key。
- 对比取舍：动态 SQL 灵活但可读性差、难做静态检查，复杂场景可考虑 QueryDSL 或 Criteria 但会失去 SQL 可控性。

**追问预判**

- 为什么用 where 而不是 where 1=1 → 语义清晰、能利用索引且不产生恒真条件，部分优化器对 1=1 处理不好。
- foreach 的 collection 报错怎么排查 → 看方法参数是否加了 @Param、类型是数组还是集合、是否被包装成 Map。
- 动态 SQL 能防注入吗 → 只要值用 `#{}` 就安全，`${}` 仍然危险。

**常见错误答法**

- 用 `status == 'Y'` 比较字符串（OGNL 把单引号当 char，结果恒不等）。
- 在 foreach 里用 `${}` 拼值（引入注入风险）。

**记忆锚点**

if 条件 → where/set 修剪 → foreach 拼集合 → bind 复用变量 → 值永远 `#{}`。

---

### 12. 【进阶】SQL 安全：批量 IN 查询与批量插入怎么写才不炸？

**口述版**

IN 列表要控大小，MySQL 虽然不像 Oracle 有 1000 的硬限制，但列表太长会导致 SQL 文本巨大、解析变慢、执行计划退化，一般按 500 到 1000 一批循环查，再在内存里合并。批量插入有两条路：一条是 foreach 拼成 `insert into t(...) values (...),(...),(...)`，一次网络往返，速度快；另一条是用 BatchExecutor 加 JDBC batch，配合 MySQL 驱动的 rewriteBatchedStatements=true 让驱动改写成多 values。要注意单条 SQL 太长会受 max_allowed_packet 限制、锁范围变大、主从延迟增加，所以还要分批提交。另外 BATCH 下 selectKey 会逐条执行，自增主键要用 useGeneratedKeys 回填。

**短版**

IN 按 500-1000 分批；批量插入用多 values 或 BATCH + rewriteBatchedStatements；控制单条 SQL 长度、分批提交；BATCH 下别用 selectKey，用 useGeneratedKeys。

**展开版**

- 机制链：foreach 生成多组占位符 → 一条 SQL 多行插入；BATCH 则 addBatch 累积 → flush 时驱动改写 → 一次发送。
- 关键结构：ForEachSqlNode、ExecutorType.BATCH、Jdbc3KeyGenerator、MySQL 的 max_allowed_packet。
- 触发条件：数据导入、定时同步、大批量状态更新。
- 例外：Oracle 的 IN 有 1000 限制；某些驱动不支持批量返回自增主键，要回查。
- 对比取舍：多 values 一条 SQL 最快但长度不可控；BATCH 内存更平稳、可分片提交，但行数与主键回填受限。

**追问预判**

- 批量插入为什么比循环单条快 → 减少网络往返和 SQL 解析次数，事务提交次数也少。
- 批处理失败怎么定位 → 整批回滚后缩小批次二分，或开驱动日志看具体哪条失败。
- 分批大小怎么定 → 看单行大小、max_allowed_packet、事务时长与主从延迟，压测后取折中值。

**常见错误答法**

- 一次拼几万条 values（SQL 过长、可能直接报错）。
- 在 BATCH 循环里用 selectKey 查主键（退化成逐条查询）。

**记忆锚点**

IN 分批 500-1000 → 多 values 或 BATCH → rewriteBatchedStatements → 控长度 + useGeneratedKeys。

---

### 13. 【兜底】SQL 安全：哪些场景必须用 `${}`，怎么用才安全？

**口述版**

必须用 `${}` 的都是 PreparedStatement 不支持占位符的位置：动态表名，比如按月份或租户分表；排序列和排序方向；动态查询列；少量枚举型标识符。安全做法是绝不把原始用户输入直接拼进去，而是做白名单映射：前端传 sortKey，服务端用 Map 映射到真实列名，映射不到就拒绝；排序方向只允许 ASC 和 DESC 两个常量；表名后缀用取模结果或正则 `^[a-zA-Z0-9_]+$` 校验；租户分表用登录态里的租户 id，不接受前端传。再叠加数据库账号最小权限、审计日志，把风险控制在可接受范围。

**短版**

动态表名、排序列/方向、动态列、枚举标识符才用；白名单映射 + 常量枚举 + 正则校验 + 最小权限，绝不直接拼用户输入。

**展开版**

- 机制链：请求参数 → 服务端映射/校验 → 拼进 SQL 片段 → 执行；校验失败直接抛业务异常。
- 关键结构：白名单 Map、枚举、正则、表名路由函数（分表后缀）、`@Param` 配合 `${}`。
- 触发条件：分库分表中间件、多租户、报表动态列、通用查询组件。
- 例外：即便做了正则，也建议再套一层白名单，正则容易被绕过或漏配。
- 对比取舍：完全禁用 `${}` 会失去分表与动态排序能力；放开又必须承担审计与代码评审成本。

**追问预判**

- 用 `#{}` 传表名会怎样 → 变成字符串字面量，SQL 语法错误或查错表。
- order by 能不能用参数化 → 不能，只能拼标识符，所以必须白名单。
- 分表场景怎么防注入 → 后缀由服务端计算，前端只传业务维度，不传表名。

**常见错误答法**

- 认为做了关键字过滤就安全（过滤可绕过，白名单才可靠）。
- 把用户输入的列名直接 `${}` 拼接（典型注入入口）。

**记忆锚点**

标识符才用 `${}` → 白名单映射 → 方向常量 → 表名服务端算 → 最小权限兜底。

---

### 14. 【进阶】SQL 安全：MyBatis 项目里 SQL 注入的真实入口有哪些？

**口述版**

真实入口比想象的多。最常见是 `'%${kw}%'` 这种 like 拼接，其次是 order by 后面直接拼前端传的字段，还有 in 列表用 `${}` 拼、动态表名拼、动态列拼。注解方式里 `@Select("... ${col} ...")` 同样危险。插件和分页组件改写 SQL 时如果拼接了外部输入，也会引入注入。另外存储过程里动态执行 SQL、拼接的 DDL 也要注意。防御不是靠过滤关键字，而是值一律 `#{}`、标识符一律白名单、数据库账号最小权限、上线前做注入扫描和代码评审规则，把 `${}` 在代码库里做成可检索、需审批的写法。

**短版**

like 拼接、order by 拼接、in 拼接、动态表名列名、注解里的 `${}`、插件改写 SQL；值用 `#{}`、标识符白名单、最小权限、代码扫描。

**展开版**

- 机制链：外部输入 → 进入 `${}` 或字符串拼接 → 成为 SQL 文本 → 数据库解析执行 → 语义被改写。
- 关键结构：`${}` 出现点、注解 SQL、插件中的 BoundSql 改写、Provider 里的字符串拼接。
- 触发条件：搜索、排序、导出、报表、通用查询组件、多租户路由。
- 例外：MyBatis 自身不会对 `${}` 做任何转义，框架层没有兜底。
- 对比取舍：全量禁用 `${}` 影响灵活性；放开就必须配套白名单、审计与自动化扫描，成本前移。

**追问预判**

- 预编译能防所有注入吗 → 只能防值位置的注入，标识符拼接仍要靠白名单。
- 怎么在 CI 里防 → 静态扫描 `${}` 出现位置、评审规则、单元测试覆盖恶意输入。
- 已经上线怎么排查 → 搜 `${}`、看慢日志里的异常 SQL、做渗透测试。

**常见错误答法**

- 只盯着登录框，忽略 order by、动态表名（入口不全）。
- 依赖 WAF 作为唯一防线（纵深不足）。

**记忆锚点**

值走 `#{}` → 标识符白名单 → 留意插件与注解 → 最小权限 + 扫描。

---

## 四、插件与分页

### 15. 【必答】插件与分页：MyBatis 插件能拦什么，分页插件怎么工作？

**口述版**

插件只能拦四类对象：Executor 的 query、update、flushStatements，StatementHandler 的 prepare、parameterize、batch、update、query，ParameterHandler 的 setParameters，ResultSetHandler 的 handleResultSets。分页插件就是拦 Executor.query：调用 startPage 时把分页参数放进 ThreadLocal，拦截器在执行前取出，先生成并执行一条 count SQL 得到总数，再把原 SQL 改写成带 limit 的语句，执行完把结果封装成分页对象。这里有两个坑，ThreadLocal 用完必须清理，否则线程池复用会串页；count SQL 要能正确去掉 order by、处理 group by 和 distinct，否则总数算错。

**短版**

四类对象固定拦截点；分页插件拦 Executor.query，ThreadLocal 取参数 → count SQL → 改写 limit → 封装结果；记得清理 ThreadLocal、正确处理 count。

**展开版**

- 机制链：startPage 写 ThreadLocal → 拦截 query → 取分页参数 → 生成 count SQL 执行 → 改写原 SQL 加 limit → 执行 → 包装 Page。
- 关键结构：PageInterceptor、Dialect（MySQL 用 limit）、Page、PageInfo、BoundSql 的 SQL 与参数映射。
- 触发条件：任何需要物理分页的列表查询；逻辑分页（RowBounds）不 rewrite，只丢弃结果。
- 例外：count 语句含 group by 时要包一层 select count(1) from (原 SQL)；union、子查询、having 都要特殊处理。
- 对比取舍：物理分页减少传输但多一次 count 查询；可配 count 缓存或近似总数；RowBounds 逻辑分页在大数据量下会拖垮内存。

**追问预判**

- PageHelper 的 ThreadLocal 什么时候清 → 拦截器执行完在 finally 清理，但嵌套调用或异常路径要自己确认。
- 分页和一对多映射能一起用吗 → 不能直接对 join 结果分页，会按行数截断，必须先分页主表再查子表。
- count 很慢怎么优化 → 简化 join、去掉 order by、加覆盖索引、缓存总数或改用近似值。

**常见错误答法**

- 说插件能拦任意方法（范围错）。
- 把 RowBounds 当成物理分页（概念混淆，它只是内存过滤）。

**记忆锚点**

四类对象固定点 → 分页拦 query → ThreadLocal 取参 → count + limit → 清理防串页。

---

### 16. 【必答】插件与分页：深分页为什么慢，怎么优化？

**口述版**

`limit 1000000, 20` 慢是因为 MySQL 要先扫描并丢弃前一百万行，再取二十行，回表次数和扫描量都随偏移量线性增长。优化有几种思路。第一是延迟关联，先用覆盖索引查出这二十条的主键，再拿主键回表取整行，减少回表量。第二是游标分页，也就是 seek 方式，前端传上一页最后一条的 id，SQL 写成 `where id > lastId order by id limit 20`，每页都能走索引定位，代价是不能随机跳页。第三是做产品约束，限制最大页数或者只允许按时间范围筛选。另外 count 查询在深分页场景也很贵，可以缓存总数、用近似值或干脆不展示总页数。

**短版**

limit 大偏移要先扫后丢；优化用延迟关联（先查主键再回表）、游标 seek 分页（where id > lastId）、限制页深或缩小范围；count 也要优化。

**展开版**

- 机制链：offset 越大 → 扫描行数越多 → 回表与排序成本上升 → 响应变慢。
- 关键结构：覆盖索引、主键回表、`where id > ? order by id limit ?`、explain 的 rows 与 type。
- 触发条件：后台列表、日志查询、导出、按时间倒序的流水。
- 例外：数据量小或 offset 小于几千时优化收益有限；带复杂过滤条件时游标分页需要联合游标（时间 + id）。
- 对比取舍：延迟关联改动小但仍有扫描；游标分页最快但不支持跳页；产品限制最简单但影响体验。

**追问预判**

- 为什么游标分页不能跳页 → 每页依赖上一页的最后一个值，没有全局偏移概念。
- 联合排序怎么做游标 → 用 `(create_time, id) < (?, ?)` 这种元组比较，或加复合索引。
- 延迟关联为什么快 → 子查询只查主键走覆盖索引，回表只针对最终二十行。

**常见错误答法**

- 只答"加索引"（深分页主因是偏移扫描，不是缺索引）。
- 认为把 limit 改小就能解决（offset 没变，扫描量不变）。

**记忆锚点**

offset 扫描 → 延迟关联 → 游标 seek → 限制页深 → count 单独优化。

---

### 17. 【进阶】插件与分页：自定义插件改写 SQL 有哪些风险？

**口述版**

最大的风险是只改了 SQL 文本，没同步参数映射。BoundSql 里 SQL 和 ParameterMapping 是一一对应的，你加了一个 `?` 却没加映射，或者删了条件没删映射，执行时就会参数错位或越界。其次是改写要考虑 SQL 形态，数据权限插件加条件时，遇到 union、子查询、group by、distinct，条件加错层级就会改变语义，比如把租户条件加到外层导致子查询没限制。第三是顺序问题，多个插件都改同一段 SQL，谁先谁后结果不同。正确做法是优先用成熟插件，必须自己写时用 JSqlParser 做语法解析而不是正则替换，改完同步维护 parameterMappings，并且覆盖多种 SQL 形态做单测。

**短版**

SQL 与 ParameterMapping 必须同步改；union/子查询/group by 下加条件要选对层级；多插件顺序影响结果；用解析器而非正则，补单测。

**展开版**

- 机制链：拦截 query → 取 BoundSql → 解析或拼接 SQL → 同步维护 parameterMappings 与 additionalParameters → 交给 StatementHandler。
- 关键结构：BoundSql、ParameterMapping、MetaObject、JSqlParser、MyBatis-Plus 的 TenantLineInnerInterceptor 可参考。
- 触发条件：多租户、数据权限、逻辑删除、SQL 审计、分页。
- 例外：只做日志和耗时统计的插件不改 SQL，风险最低；改 SQL 的插件必须处理 count 语句。
- 对比取舍：插件无侵入但隐式、难测试；在业务层显式加条件可控但重复代码多。

**追问预判**

- 改了 SQL 为什么参数对不上 → 只改文本没同步 ParameterMapping，占位符与映射数量或顺序错位。
- 数据权限条件加在哪一层 → 加在内层子查询或原表上，不能只加在外层，否则绕过限制。
- count 语句要不要一起改 → 要，分页或权限插件必须保证 count 与原查询口径一致。

**常见错误答法**

- 用字符串 replace 改 SQL 且不同步参数（运行时参数错位）。
- 认为插件改写只在简单查询生效（忽略 count、union 等形态）。

**记忆锚点**

SQL 改 → 参数映射同步 → 选对层级 → 顺序可控 → 解析器 + 单测。

---

### 18. 【兜底】插件与分页：多个插件顺序冲突怎么排查和解决？

**口述版**

先看注册顺序。MyBatis 的 InterceptorChain 按 Configuration 添加顺序逐个包装，后加的包在外层、先执行，所以租户插件和分页插件谁先谁后会直接决定 SQL 形态。排查时我会在插件里打印入参和最终 BoundSql，看每一层看到的 SQL 长什么样；也可以断点看 Invocation 的 target 是哪个代理；再确认 Spring Boot 下是通过 ConfigurationCustomizer 还是 mybatis.configuration.interceptors 注册的，Bean 的 @Order 未必影响 MyBatis 链。解决上，优先合并成单个插件或使用官方组合，其次显式控制添加顺序，让租户条件先加、分页最后包 limit，并针对组合场景补集成测试。

**短版**

后注册在外层先执行；打印每层 BoundSql、看 Invocation.target、确认注册方式；显式控制顺序（租户先、分页后）或合并插件，补组合测试。

**展开版**

- 机制链：注册列表顺序 → pluginAll 依次 wrap → 外层先执行 → 内层看到已改写的 SQL。
- 关键结构：InterceptorChain、Plugin 的 InvocationHandler、Spring Boot 的 ConfigurationCustomizer / mybatis.configuration.interceptors。
- 触发条件：同时使用分页、多租户、数据权限、逻辑删除等多个改写型插件。
- 例外：只读统计类插件顺序影响小；改 SQL 的插件顺序错误会直接产生错误结果。
- 对比取舍：显式排序简单但依赖配置纪律；合并插件可控但代码耦合、复用性下降。

**追问预判**

- @Order 能控制 MyBatis 插件顺序吗 → 不能可靠控制，MyBatis 按 Configuration 添加顺序包装，Bean 顺序不等于添加顺序。
- 分页插件应该在最外层还是最内层 → 一般最后执行，确保 limit 包在最终 SQL 外面，租户条件先进子查询。
- 怎么验证顺序正确 → 看最终 SQL 文本和 count 语句，跑组合场景的集成测试。

**常见错误答法**

- 用 @Order 调整后不验证最终 SQL（假设生效）。
- 把 count SQL 交给内层插件处理（顺序导致的漏改）。

**记忆锚点**

添加顺序决定包裹 → 后注册先执行 → 打印 BoundSql 定位 → 租户先分页后 → 集成测试验证。

---

## 五、映射与 ORM 取舍

### 19. 【必答】映射：ResultMap、自动映射和 association/collection 怎么配？

**口述版**

resultType 走自动映射，列名和属性名一致就能填，开 mapUnderscoreToCamelCase 后下划线也能对上，适合单表简单查询。resultMap 用于列名和属性不一致、需要类型转换、或者要映射关联对象的情况：association 处理一对一，collection 处理一对多，discriminator 按列值走不同映射。关联映射有两种写法，嵌套结果是 join 一次查回来再按 id 组装，需要配 `<id>` 让 MyBatis 知道怎么去重；嵌套查询是先查主表再逐条触发子查询，写起来简单但会 N+1。前者快但要处理数据放大和分页失效，后者清晰但必须配合延迟加载或批量查询。

**短版**

resultType 自动映射、驼峰开关；resultMap 处理不一致与关联；association 一对一、collection 一对多；嵌套结果 join 一次查回、需 id 去重；嵌套查询 N+1。

**展开版**

- 机制链：ResultSetHandler → 按 ResultMap 逐列取值 → TypeHandler 转换 → 创建对象 → association/collection 递归组装 → 按 id 合并。
- 关键结构：ResultMap、id/result、association、collection、discriminator、autoMappingBehavior、typeHandler。
- 触发条件：多表关联、枚举转换、JSON 字段、继承结构。
- 例外：嵌套结果配分页时，limit 作用在行数上，会导致主表记录被截断，必须先分页主表。
- 对比取舍：嵌套结果一次网络往返但 SQL 复杂、结果集放大；嵌套查询 SQL 简单但 N+1，需要延迟加载或二次批量查询。

**追问预判**

- collection 为什么只返回一条 → 少了 `<id>` 或 id 列重复，MyBatis 无法区分聚合边界。
- 自动映射会被 resultMap 影响吗 → 配了 resultMap 后受 autoMapping 控制，复杂映射建议显式写全。
- 枚举怎么映射 → 用 EnumTypeHandler 存名字或 EnumOrdinalTypeHandler，推荐自定义存业务码。

**常见错误答法**

- 用 resultType 做一对多映射（结构对不上，只能拿到一条）。
- 嵌套结果 join 后直接分页（行数被截断）。

**记忆锚点**

resultType 自动 → resultMap 手动 → association/collection → join 要 id 去重 → 分页先主表。

---

### 20. 【必答】映射：N+1 问题怎么产生、怎么定位和解决？

**口述版**

N+1 是主查询返回 N 条记录，每条又触发一次关联查询，总共 N+1 次数据库交互。MyBatis 里主要出现在 association 或 collection 用嵌套 select 的场景，或者业务代码在循环里逐条查详情。定位看 SQL 日志最直接，同一个模板 SQL 重复出现几十上百次，或者 APM 里单接口 SQL 次数异常高。解决方式有三类：改成 join 加嵌套结果一次查回，但要注意数据放大和分页；用二次批量查询，先拿主键集合再 `in` 一次查子表，在内存里分组；或者加缓存减少重复查询。我的默认选择是列表页用批量查询，详情页用 join，避免循环里查库。

**短版**

主查询 N 条、每条再查一次；嵌套 select 或循环查库导致；看 SQL 日志/APM 定位；解决用 join 嵌套结果、二次 IN 批量查、缓存。

**展开版**

- 机制链：主查询返回列表 → 遍历每条 → 触发嵌套 select → 单条 SQL 执行 N 次。
- 关键结构：嵌套查询的 select 属性、fetchType、lazyLoadingEnabled、`in` 批量查询、Map 分组。
- 触发条件：列表接口、导出、消息消费里逐条补数据。
- 例外：N 很小（个位数）且命中缓存时影响有限；延迟加载把 N+1 推迟到序列化阶段，问题更隐蔽。
- 对比取舍：join 一次往返但结果集放大；批量 IN 两次往返、内存分组，最平衡；缓存最快但有失效成本。

**追问预判**

- 为什么开了延迟加载反而更容易出问题 → 序列化或 toString 触发加载，此时 SqlSession 可能已关闭。
- 分页 + 一对多怎么避免 N+1 → 先分页主表拿 id，再用一次 IN 查子表组装。
- 怎么在测试里卡住 N+1 → 统计单接口 SQL 条数、加断言，或引入 p6spy 做基线。

**常见错误答法**

- 只用 join 解决所有场景（忽略分页与数据放大）。
- 认为 N+1 只和 MyBatis 有关（JPA 的懒加载同样有）。

**记忆锚点**

N 条主 + N 次子 → 嵌套 select/循环 → 日志数 SQL → join 或 IN 批量 → 分页先主表。

---

### 21. 【进阶】映射：延迟加载的原理和坑？

**口述版**

延迟加载靠代理实现，查到主对象后，关联属性先塞一个 CGLIB 或 JAVASSIST 代理，只有调用 getter 时才真正发 SQL 加载。开关是 lazyLoadingEnabled，aggressiveLazyLoading 在 3.4.1 之后默认 false，也就是按需加载而不是加载任一属性就全加载。坑主要有几类：JSON 序列化会遍历所有 getter，等于把延迟加载全部触发，N+1 在序列化阶段爆发；toString、equals 同样会触发；如果 SqlSession 已经关闭，代理再去查库会报会话已关闭；跨线程使用代理对象也会因为上下文丢失失败。所以我的实践是列表和对外接口不用懒加载，明确用 DTO 组装，只在内部聚合场景按需开启并配 fetchType 覆盖。

**短版**

CGLIB/JAVASSIST 代理，调 getter 才查库；aggressiveLazyLoading 默认 false；序列化/toString 触发加载、会话关闭报错、跨线程失效，接口层建议关掉用 DTO。

**展开版**

- 机制链：结果映射创建代理对象 → setter 拦截 → 首次 get 触发 ResultLoader → 用原 SqlSession 执行嵌套查询。
- 关键结构：ProxyFactory（CGLIB/JAVASSIST）、ResultLoaderMap、lazyLoadingEnabled、aggressiveLazyLoading、fetchType。
- 触发条件：嵌套查询、关联对象较大且不一定使用；聚合内部导航。
- 例外：`fetchType="eager"` 可单条覆盖全局；基本类型和已加载属性不生成代理。
- 对比取舍：按需查库省流量，但把 SQL 时机变得不可预测，调试和性能分析都更难。

**追问预判**

- 为什么接口返回时才报错 → 序列化触发 getter，此时事务已结束、SqlSession 已关闭。
- 怎么判断有没有触发 → 看日志里嵌套 SQL 出现的时机，或断点 ResultLoader。
- 能不能彻底关掉 → 全局 lazyLoadingEnabled=false，嵌套查询就立即执行，等于放弃延迟。

**常见错误答法**

- 认为延迟加载一定能减少查询次数（触发不当会变成 N+1）。
- 在 Controller 层返回实体还指望懒加载安全（会话已关）。

**记忆锚点**

代理占位 → getter 触发 → 序列化是雷区 → 会话关闭报错 → 接口层用 DTO。

---

### 22. 【进阶】映射：MyBatis 与 JPA/Hibernate 怎么选？

**口述版**

本质是 SQL 可控性和开发效率的取舍。MyBatis 把 SQL 交给开发者，执行计划、索引、分页、批量都能精确控制，适合复杂查询、报表、高并发核心链路和已有 SQL 规范的团队，缺点是重复劳动多、简单 CRUD 也要写映射。JPA 胜在对象导航、自动 CRUD、跨数据库和领域建模，开发快，但复杂查询要靠 JPQL、Criteria 或原生 SQL，N+1、懒加载、脏检查、flush 时机都是常见坑，批量写入也偏弱。我的选型是核心交易和高并发查询用 MyBatis，后台管理、领域模型清晰的模块用 JPA，同一项目里也可以混用，但要有清晰边界，别让两套映射同时写同一张表。

**短版**

MyBatis SQL 可控、适合复杂查询与高并发；JPA 开发快、适合领域模型与 CRUD，但 N+1/懒加载/批量弱；按模块选，可混用但边界要清晰。

**展开版**

- 机制链：MyBatis 手写 SQL → 精确控制执行计划；JPA 由持久化上下文管理实体 → 脏检查生成 SQL。
- 关键结构：SqlSession/Executor 对比 EntityManager/持久化上下文、一级缓存、flush 模式。
- 触发条件：团队 SQL 能力、查询复杂度、是否跨数据库、是否需要快速迭代。
- 例外：JPA 也能写原生 SQL 和投影，MyBatis 也能用代码生成器降低重复劳动。
- 对比取舍：MyBatis 可控但开发成本高；JPA 效率高但性能问题隐式、调优门槛高。

**追问预判**

- JPA 的 N+1 怎么解决 → fetch join、EntityGraph、批量抓取，或改 DTO 投影。
- 为什么 JPA 批量插入慢 → 默认逐条 insert、flush 时机与自增主键导致无法批处理，需要配 batch_size 和 ID 生成策略。
- 混用要注意什么 → 事务与连接要统一、缓存别重复、实体与表归属要划清。

**常见错误答法**

- 说 JPA 不能写复杂 SQL（原生 SQL、Criteria 都支持）。
- 认为 MyBatis 性能一定比 JPA 好（取决于 SQL 和用法）。

**记忆锚点**

MyBatis 控 SQL → JPA 控模型 → 核心链路偏 MyBatis → CRUD 偏 JPA → 混用划边界。

---

### 23. 【兜底】映射：ORM 与手写 SQL 的边界在哪，读写模型要不要拆？

**口述版**

我的划分是写模型用 ORM 或聚合根保证一致性，读模型用手写 SQL 直接投影 DTO。写侧关心的是业务规则、事务边界和聚合完整性，用对象建模更自然；读侧关心的是查询性能和返回结构，往往要跨多个聚合、做聚合统计，用 ORM 导航既低效又难控。所以列表、报表、大屏这类读多写少的场景，直接写 SQL 返回 DTO，不要先查实体再转换。读写拆不拆要看复杂度，简单系统共用一套模型就够，一旦出现读写字段差异大、查询性能要求高、多团队并行，就值得做 CQRS 式的分离。物理库可以先不拆，逻辑上把 Command 和 Query 的入口分开。

**短版**

写用 ORM 保一致性，读用手写 SQL 投影 DTO；列表报表别查实体再转；复杂度低不拆，读写差异大或性能要求高再逻辑拆 CQRS。

**展开版**

- 机制链：写请求 → 聚合校验 → 事务提交；读请求 → 查询服务 → SQL 投影 → DTO 返回。
- 关键结构：聚合根、Repository、QueryService、DTO、读写分离数据源。
- 触发条件：读写比例悬殊、查询跨多个聚合、字段权限与脱敏要求不同。
- 例外：强一致读（写完立刻读）要走主库或同事务，不能直接切从库。
- 对比取舍：拆分提升读性能与模型清晰度，代价是代码量、同步延迟与一致性处理成本上升。

**追问预判**

- 读写分离后写完立刻读怎么办 → 写后读主、会话粘滞，或等主从位点。
- DTO 投影会不会丢复用 → 读模型本来就是按场景定制的，复用应放在查询组合层而非实体层。
- 什么时候不该拆 → 单体早期、查询简单、团队规模小，拆了只会增加维护成本。

**常见错误答法**

- 所有查询都先查实体再转 DTO（多余开销、易触发懒加载）。
- 一上来就物理分库做 CQRS（过度设计）。

**记忆锚点**

写管一致性 → 读管性能 → 读直接投影 DTO → 逻辑先拆 → 物理后拆。

---

## 六、事务与工程治理

### 24. 【必答】事务：MyBatis 事务和 Spring 事务怎么衔接？

**口述版**

衔接点是 SqlSessionTemplate 和 DataSourceTransactionManager。开启 `@Transactional` 后，Spring 从数据源拿连接并绑定到当前线程，MyBatis 的 SqlSessionTemplate 在执行时通过 SqlSessionInterceptor 去 TransactionSynchronizationManager 找已绑定的 SqlSession，找到就复用，没找到才新建并在用完后关闭。所以同一个事务里所有 Mapper 调用共用一条连接和一个会话，提交回滚由 Spring 统一控制，MyBatis 自己不 commit。要注意事务只对 RuntimeException 和 Error 默认回滚，受检异常要显式配 rollbackFor；自调用不走代理导致事务失效；多数据源要各自配事务管理器；SqlSession 和事务上下文都不能跨线程。

**短版**

Spring 管连接并绑定线程，SqlSessionTemplate 复用事务内会话，提交回滚由 Spring 控制；注意 rollbackFor、自调用失效、多数据源事务管理器、不能跨线程。

**展开版**

- 机制链：@Transactional → 事务管理器取连接绑定线程 → Mapper 调用 → Template 取到同一 SqlSession → 业务执行 → 提交/回滚 → 释放连接。
- 关键结构：DataSourceTransactionManager、SqlSessionTemplate、SqlSessionInterceptor、TransactionSynchronizationManager。
- 触发条件：所有需要原子性的写操作；跨多个 Mapper 的操作必须在同一事务内。
- 例外：REQUIRES_NEW 会挂起当前事务、另取连接，注意连接池容量与锁；只读事务可做优化但不能保证不写。
- 对比取舍：声明式事务简洁但边界隐式，长事务会占连接、放大锁范围，需要在服务层控制粒度。

**追问预判**

- 为什么 self-invocation 事务不生效 → 调用没经过代理对象，注解不被解析。
- 捕获异常后事务还回滚吗 → 默认不会，除非重新抛出或手动 setRollbackOnly。
- MyBatis 自己 commit 会怎样 → 会破坏 Spring 事务边界，导致部分提交，必须交给 Spring。

**常见错误答法**

- 在 Mapper 或 DAO 上开事务、在 Service 外再开一层（边界混乱）。
- 认为受检异常也会自动回滚（默认只回滚运行时异常）。

**记忆锚点**

Spring 绑连接 → Template 复用会话 → 提交归 Spring → rollbackFor 显式配 → 别跨线程。

---

### 25. 【进阶】事务：批量写入怎么优化，BATCH 有哪些限制？

**口述版**

优化分三层。SQL 层用多 values 插入，一次往返写多行；驱动层开 MySQL 的 rewriteBatchedStatements，让 JDBC batch 被改写成多 values；框架层用 ExecutorType.BATCH，循环里只 addBatch，最后 flushStatements 统一提交。再往外是事务与批次控制，按五百到一千条一批提交，避免超大事务导致 undo、binlog、锁和主从延迟一起放大。BATCH 的限制要说清楚：影响行数返回不准，通常是 -2；selectKey 会逐条执行，自增主键要用 useGeneratedKeys；批量里某条失败整批回滚，定位要靠缩小批次或驱动日志；另外连接池要留够余量，批量任务别把在线流量挤掉。

**短版**

多 values + rewriteBatchedStatements + ExecutorType.BATCH；按 500-1000 分批提交；BATCH 行数不准、selectKey 退化、失败整批回滚，用 useGeneratedKeys 和独立线程池。

**展开版**

- 机制链：循环 addBatch → 累积到阈值 → flushStatements → 驱动改写 → 一次网络发送 → 分批提交。
- 关键结构：BatchExecutor、Jdbc3KeyGenerator、rewriteBatchedStatements、max_allowed_packet、连接池隔离。
- 触发条件：数据导入、批量状态更新、离线同步、消息批量落库。
- 例外：Oracle 的批量语法与 MySQL 不同；部分云数据库对单条 SQL 大小有限制。
- 对比取舍：批越大吞吐越高但内存、锁、回滚成本越大；独立连接池能隔离影响但增加资源占用。

**追问预判**

- 批处理里怎么拿自增主键 → useGeneratedKeys + keyProperty，依赖驱动支持；不支持就分批回查。
- 为什么批量任务会拖垮在线接口 → 共用连接池和数据库资源，长事务占连接、锁竞争。
- 批量更新有唯一键冲突怎么办 → insert on duplicate key update，或先查后分拣，注意并发下的竞态。

**常见错误答法**

- 循环里逐条 insert 还抱怨慢（没做批处理）。
- 一次提交十万条（超大事务，失败代价高）。

**记忆锚点**

多 values → 驱动改写 → BATCH + flush → 分批提交 → 独立池隔离。

---

### 26. 【兜底】事务：线上 SQL 变慢，怎么定位到具体语句和原因？

**口述版**

先确认是 SQL 慢还是等资源。看接口耗时分布和连接池指标，如果拿连接就花了很久，那是池子或长事务问题，不是 SQL 本身。确认是 SQL 后，开 MyBatis 的 Mapper 日志或上 p6spy、Druid 监控，拿到真实执行语句和参数，注意日志里的 SQL 和实际执行的可能因插件改写而不同。然后对慢语句做 explain，重点看 type、key、rows、filtered 和 Extra，常见原因是缺索引、索引失效、回表过多、filesort、临时表、深分页、N+1。再结合慢查询日志和 pt-query-digest 做聚合排序，找到 Top SQL。优化后要回归压测，确认执行计划和耗时都符合预期。

**短版**

先分清等连接还是 SQL 慢；Mapper 日志/p6spy 拿真实语句与参数；explain 看 type/key/rows/Extra；慢日志聚合找 Top SQL；优化后压测回归。

**展开版**

- 机制链：接口耗时 → 链路分段（池等待 / SQL 执行 / 结果映射）→ 定位语句 → explain → 改索引或 SQL → 回归。
- 关键结构：MyBatis 日志、p6spy、Druid StatFilter、slow_query_log、performance_schema、explain analyze。
- 触发条件：接口 P99 升高、数据库 CPU 高、主从延迟、连接池打满。
- 例外：结果集过大时耗时在映射和网络传输，不在数据库执行；插件改写导致的慢 SQL 要看最终 BoundSql。
- 对比取舍：加索引见效快但影响写入与空间；改 SQL 更治本但成本高；缓存能救急但要注意一致性。

**追问预判**

- explain 里哪些信号最危险 → type 为 ALL、key 为空、rows 很大、Extra 出现 filesort 或 temporary。
- 索引明明有为什么不走 → 隐式转换、函数包裹列、最左前缀不满足、区分度低、优化器估算偏差。
- 怎么区分是锁等待 → 看 innodb_lock_wait_timeout、performance_schema 的等待事件和锁信息。

**常见错误答法**

- 一上来就加索引（没先确认慢在 SQL 还是资源等待）。
- 只看平均耗时，不看 P99 和具体 SQL（定位不到长尾）。

**记忆锚点**

先分池还是 SQL → 日志拿真语句 → explain 四看 → 慢日志聚 Top → 压测回归。

---

