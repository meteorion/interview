# Oracle 核心知识点学习文档

Nov 23, 2026 · @meteorzhong

## 导读

本文按「定位 → 模型 → 流程 → 机制 → 性能 → 可靠性 → 生态」的主线整理 Oracle，重点放在**面试能讲清原理、线上能定位问题**两条线上。沿用其他文档的三级标注：**【核心】** 面试必问，要能讲清原理和「为什么」；**【重点】** 常被追问，要能说清机制与取舍；**【了解】** 知道概念即可。默认版本为 Oracle 19c，括号内标注差异的属于 12c / 23ai 特性。

| 大纲节点 | 级别 | 本文章节 |
| --- | --- | --- |
| 定位与边界 | 重点 | 一、定位与边界 |
| 核心模型（实例与 SGA/PGA、逻辑存储） | 核心 | 二、核心模型 |
| 生命周期与流程（SQL 解析、redo 与检查点、库内调度） | 核心 | 三、生命周期与流程 |
| 关键机制（一致性读、事务、锁、索引类型、存储程序） | 核心 | 四、关键机制 |
| 性能与容量（CBO、绑定变量、高级 SQL、空间与内存） | 核心 | 五、性能与容量 |
| 可靠性与排障（AWR/ASH、阻塞、备份与闪回） | 核心 | 六、可靠性与排障 |
| 生态与演进（RAC、Data Guard、多租户、数据同步） | 重点 | 七、生态与演进 |

**复习顺序**：实例与内存 → 逻辑存储 → 启动与解析 → redo 与检查点 → 一致性读与事务 → 锁 → 索引与执行计划 → CBO 与绑定变量 → 空间与内存 → AWR/ASH 与阻塞 → 备份与闪回 → RAC/Data Guard/多租户。如果只准备支付、金融类岗位，优先级最高的是**一致性读、redo 与提交、锁与阻塞排查、备份恢复**这四块。

**和 MySQL 文档的关系**：MySQL 那篇讲的是 InnoDB 的页、MVCC、两阶段提交；Oracle 的对应概念是块、SCN + undo 构造 CR 块、redo 前滚加回滚。建议对照着看，面试里「Oracle 和 MySQL 事务实现有什么不同」是高频题。

## 一、定位与边界

Oracle 是典型的**共享磁盘 + 多进程**商业数据库，长期占据银行、保险、电信、政务的核心交易系统。它的设计目标不是「轻量易用」，而是「大事务量下的稳定性、可预测的恢复能力、以及完整的企业级工具链」。

### 1.1 Oracle 在技术栈里的位置【重点】

- **强项**：事务一致性与崩溃恢复经过几十年验证；RAC + Data Guard 提供同城高可用与异地容灾的完整方案；AWR/ASH/SQL Monitor 的诊断体系至今仍是商业库里最完整的；PL/SQL 让重计算可以下推到库内。
- **代价**：授权按 CPU/用户数计费，Diagnostics Pack、Tuning Pack、RAC、Multitenant、In-Memory、Active Data Guard 都是**单独收费的选项**；运维门槛高（内存、undo、归档、备份策略都要人管）；生态偏封闭，云原生适配弱。
- **典型边界**：核心账务、清结算、批量对账这类「数据不能错、故障要能恢复到秒级」的系统适合 Oracle；互联网 C 端高并发、快速迭代的业务，MySQL / PostgreSQL 的成本和弹性更合适。

**面试口径**：不要笼统说「Oracle 性能好」。正确说法是「Oracle 的强项是**一致性保证和恢复能力**，加上 RAC/DG 的高可用组合；性能优势要看场景，OLTP 单机 MySQL 也能打，Oracle 的优势在复杂查询、并行、以及 Exadata 这类一体机的卸载能力」。

### 1.2 与 MySQL / PostgreSQL 的关键差异【重点】

| 维度 | Oracle | MySQL（InnoDB） | PostgreSQL |
| --- | --- | --- | --- |
| 进程模型 | 多进程，每个连接一个 server process | 单进程多线程 | 多进程 |
| 内存结构 | SGA 共享 + PGA 私有，职责清晰 | Buffer Pool + 会话私有内存 | Shared Buffers + 每进程私有 |
| 一致性读 | SCN + undo 构造 CR 块 | 隐藏列 + undo 版本链 + ReadView | MVCC（元组多版本，旧版本留在堆表） |
| 默认隔离级别 | Read Committed（语句级一致） | Repeatable Read | Read Committed |
| 事务提交 | LGWR 刷 redo + 推进 SCN，块不清理（延迟块清除） | redo + binlog 两阶段提交 | WAL 刷盘 |
| 索引组织 | 堆表 + B*Tree 索引存 rowid | 聚簇索引（主键即表） | 堆表 + 索引存 TID |
| 高可用 | RAC（共享存储多实例）+ Data Guard | 主从复制 / MGR | 流复制 / Patroni |
| 扩展方式 | 纵向 + RAC + 分区 + 读写分离 | 分库分表为主 | 分库分表 + Citus |

### 1.3 什么场景不该用 Oracle【了解】

- 数据量小、QPS 低、团队没有专职 DBA：授权费和维护成本都不划算。
- 需要频繁弹性扩缩容、按量计费的互联网业务：云上 MySQL/PG 更合适。
- 已有大量 PL/SQL 之外的新技术栈（Kafka、Flink、湖仓）：Oracle 只能作为源库，做实时分析不如专用引擎。
- 但**迁移存量 Oracle 要非常谨慎**：PL/SQL、hint、执行计划、分区、LOB、序列、字符集语义都有坑，属于「能跑」和「敢切」之间的差距。

## 二、核心模型

Oracle 把「运行态」和「持久化数据」严格分开：**实例（Instance）= 内存 + 后台进程**，**数据库（Database）= 磁盘上的文件集合**。这是理解 Oracle 一切机制的前提。

### 2.1 实例与数据库：一组文件、一块内存【核心】

| 归属 | 组成 | 说明 |
| --- | --- | --- |
| 实例 | SGA、后台进程、参数文件 | 启动后存在，关闭即消失；一个实例同一时刻只打开一个库（RAC 例外：多实例开同一个库） |
| 数据库 | 数据文件、在线 redo 日志、控制文件 | 控制文件记录数据文件与 redo 的位置和 SCN，是「库的地图」 |
| 其他文件 | 参数文件（spfile/pfile）、密码文件、归档日志、告警日志、trace、RMAN 备份片 | 归档日志在归档模式下必需；告警日志是第一手的排障入口 |

**关键点**：`startup` 的实质是「按参数文件建内存和进程 → 读控制文件知道文件在哪 → 打开数据文件和 redo」。所以控制文件丢了要恢复，参数文件错了连实例都起不来。

### 2.2 SGA：共享内存的组成【核心】

| 组件 | 作用 | 关键点 |
| --- | --- | --- |
| Shared Pool | Library Cache（SQL 文本、解析树、执行计划）、Data Dictionary Cache、Result Cache、Reserved Pool | 硬解析在此完成；争用表现为 `library cache` 相关 latch/mutex；碎片化会报 ORA-04031 |
| Database Buffer Cache | 缓存 8 KB 数据块（默认块大小），含 default/keep/recycle 三个池 | 脏块由 DBWn 异步写；一致性读的 CR 块也在这里构造，**不落盘** |
| Redo Log Buffer | 循环使用的 redo 缓冲区 | 顺序写、LGWR 负责刷盘；小事务高频提交会把瓶颈推到这里（`log file sync`） |
| Large Pool | RMAN 备份、并行执行、共享服务器模式的 UGA、I/O 从属进程 | 大块分配放这里，避免打碎 Shared Pool |
| Java Pool / Streams Pool | JVM 与 Streams（已弃用）专用 | 用得少，知道即可 |
| Fixed SGA | latch、内部指针等固定结构 | 不参与自动调整 |

**自动管理**：`SGA_TARGET` 打开自动共享内存管理（ASMM），Oracle 按负载在 Shared Pool 与 Buffer Cache 之间调配；`MEMORY_TARGET` 更进一步把 SGA 和 PGA 一起管（AMM），但依赖 `/dev/shm`，与 HugePages 冲突，生产上更常见的是 **ASMM + HugePages**。

**为什么 buffer cache 不能无限大**：脏块越多，检查点要写的量越大，崩溃恢复时间越长；另外 buffer cache 过大挤压 Shared Pool 会引发解析争用。`V$DB_CACHE_ADVICE` 能给不同大小的预估命中率。

### 2.3 PGA：进程私有内存【重点】

PGA 是每个 server process 私有的内存，主要包括**排序区、哈希区、位图合并区、游标状态、UGA**（专用服务器模式下 UGA 在 PGA 里）。

- `WORKAREA_SIZE_POLICY=AUTO` 时由 `PGA_AGGREGATE_TARGET` 统一分配，单个操作最多用多少由 Oracle 自己算。
- 排序/哈希放不下就溢出到**临时表空间**（temp），产生多趟排序，表现为 `direct path write temp` 等待，性能急剧下降。
- 12c 起有 `PGA_AGGREGATE_LIMIT` 硬上限，超过会报 **ORA-04036**，防止连接风暴把物理内存吃光。
- 诊断视图：`V$PGASTAT`（命中率、溢出次数）、`V$PGA_TARGET_ADVICE`、`V$SQL_WORKAREA_ACTIVE`。

**SGA 与 PGA 的区别（高频题）**：SGA 所有进程共享、启动时分配、可被自动调整；PGA 每进程私有、随连接增长、不共享。所以「连接数暴涨」伤的是 PGA 和进程数，「大查询并发」伤的是 PGA 的排序区和 temp 空间。

### 2.4 后台进程分工【核心】

| 进程 | 职责 | 出问题时的现象 |
| --- | --- | --- |
| DBWn | 把脏块写到数据文件，推进增量检查点 | `db file parallel write` 高、`free buffer waits` |
| LGWR | 把 redo log buffer 顺序写到在线日志，**提交必须等它** | `log file sync`（用户侧）、`log file parallel write`（LGWR 自身） |
| CKPT | 更新数据文件头与控制文件的检查点信息，触发 DBWn | 检查点过密导致 IO 抖动 |
| SMON | 实例恢复（前滚 + 回滚）、清理临时段、合并空闲空间 | 异常关机后启动变慢 |
| PMON | 清理异常终止的进程/会话，释放锁与资源 | 会话残留、锁不释放 |
| ARCn | 日志切换后把在线日志归档 | `ARCH` 相关等待、ORA-00257 |
| MMON/MMNL | 采集 AWR 快照与度量 | 快照缺失、AWR 报告断档 |
| RECO / LMS / LMD / LCK | RAC 的分布式事务恢复与全局缓存、全局队列服务 | `gc` 系列等待 |

### 2.5 逻辑存储：表空间 → 段 → 区 → 块【核心】

```
表空间（Tablespace）
  └─ 段（Segment：一张表 / 一个索引 / 一个 undo 段 / 一个临时段）
       └─ 区（Extent：一组连续块，分配的最小单位）
            └─ 块（Block：默认 8 KB，IO 的最小单位）
```

- **块大小**在建库时由 `DB_BLOCK_SIZE` 决定，之后不能改；表空间可以指定非默认块大小（需对应 cache）。
- **表空间**分为永久（数据）、undo、临时三类；`SYSTEM` 放数据字典，`SYSAUX` 放 AWR、OEM 等辅助组件，业务数据必须独立表空间。
- **ASSM**（自动段空间管理）用位图管理块的空闲状态，取代了老式的 freelist；`PCTFREE` 控制块内为更新预留的空间比例。
- **行迁移与行链接**：行变长后原块放不下，整行搬到新块（行迁移，原块留转发地址），或一行跨多块（行链接，常见于大字段）。行迁移会让走索引的查询多一次 IO，`V$SYSSTAT` 的 `table fetch continued row` 可以观察。
- **undo 表空间**：保存前镜像，支撑回滚、一致性读、实例恢复；`UNDO_RETENTION` 控制保留时间，`RETENTION GUARANTEE` 可保证不被覆盖（代价是空间不够时报 ORA-30036）。
- **临时表空间**：排序、哈希、全局临时表的数据落在这里；临时段不产生 redo，这也是大批量操作用临时表更快的原因。

### 2.6 数据块内部结构与 ITL【重点】

一个块大致分五部分：块头（含 SCN、ITL 列表）、表目录、行目录、空闲空间、行数据。

- **ITL（Interested Transaction List）**：记录哪些事务正在修改这个块。一个事务第一次改某块时要在 ITL 里占一个槽，`INITRANS` 决定初始槽数（默认 1 或 2，最大 255）。
- 槽位不够时会动态扩展，扩展不了就等待 **`enq: TX - allocate ITL entry`**。高并发写同一张表时这是典型瓶颈，处理办法是调大 `INITRANS`（需重建表/索引）或降低并发热度。
- 块头 SCN 是一致性读判断「这个块是否比我看到的旧」的依据。

### 2.7 连接模型：专用、共享与 DRCP【了解】

- **专用服务器（Dedicated）**：一个连接一个 server process，最常见，隔离性好但连接数高时进程和 PGA 压力大。
- **共享服务器（Shared Server）**：多个会话复用少量 server process，UGA 移到 Large Pool；适合大量低频连接，不适合长事务。
- **DRCP / 连接池**：应用侧用 UCP/HikariCP 做池化，比库内共享服务器更常用。

## 三、生命周期与流程

这一章回答三个问题：实例怎么起来、SQL 怎么被解析执行、redo 和检查点怎么保证崩溃可恢复。

### 3.1 启动与关闭的三个阶段【核心】

| 阶段 | 读什么 | 能做什么 |
| --- | --- | --- |
| NOMOUNT | 参数文件 → 分配 SGA、启动后台进程 | 建库、恢复控制文件、改参数（部分） |
| MOUNT | 控制文件 → 知道数据文件和 redo 在哪 | `recover database`、改归档模式、闪回数据库 |
| OPEN | 打开数据文件与在线日志 | 正常业务访问；若上次非干净关闭，SMON 先做实例恢复 |

**关闭方式**：

- `NORMAL`：等所有会话主动断开，生产上几乎不用。
- `TRANSACTIONAL`：等当前事务结束、断开会话。
- `IMMEDIATE`：回滚未提交事务、断开连接、做检查点，**推荐方式**。
- `ABORT`：不做检查点直接退出，下次启动需要实例恢复；`STARTUP FORCE` 就是 abort + startup。

### 3.2 SQL 解析：硬解析、软解析与共享池【核心】

1. **语法检查**：SQL 文本是否符合语法。
2. **语义与权限检查**：表、列是否存在，是否有权限（查数据字典，消耗 Data Dictionary Cache）。
3. **哈希查找**：把 SQL 文本做 hash，在 Library Cache 找相同语句。文本必须**逐字符一致**（空格、大小写、注释都算），所以拼 SQL 会导致大量硬解析。
4. **命中 → 软解析**：复用已有执行计划，直接执行。
5. **未命中 → 硬解析**：CBO 计算代价、生成执行计划、写入 Library Cache。硬解析消耗 CPU 和 Shared Pool 的 latch/mutex，高并发下表现为 `library cache: mutex X`、`cursor: pin S wait on X`。

**减少硬解析的手段**：

- 用**绑定变量**（`WHERE id = :1`），让不同参数共用同一游标。
- 会话游标缓存 `SESSION_CACHED_CURSORS`，减少重复解析。
- PL/SQL 的静态 SQL 天然使用绑定变量；动态 SQL 要显式 `USING`。
- 应用层用预编译语句（JDBC `PreparedStatement`），并避免在 SQL 里拼注释、换行。

**解析之后的执行**：`open → bind → execute → fetch`。批量操作要用数组绑定（JDBC `addBatch`），否则每行一次网络往返，这是很多「库慢」问题的真实原因。

### 3.3 一条 UPDATE 的完整链路【核心】

1. **开启事务**：在 undo 表空间分配 undo 段和槽位，事务表登记，分配一个事务 ID。
2. **读块**：目标块不在 buffer cache 就从数据文件读入（`db file sequential read`）。
3. **占 ITL**：在块头 ITL 中登记本事务，同时加行级锁（TX 锁，记录在 ITL 里，不额外占内存）。
4. **写 undo**：把前镜像写入 undo 块，undo 的修改本身也产生 redo。
5. **改数据块**：修改 buffer 中的块，生成 redo change vector 写入 redo log buffer，块变脏。
6. **提交（COMMIT）**：写一条 commit 记录并推进 SCN，**LGWR 把 redo 刷到磁盘**（用户侧等待 `log file sync`），释放行锁，undo 保留（供一致性读与回滚）。
7. **不清理块**：提交时并不清理块上的 ITL 和锁信息（**延迟块清除**），下次有人访问这个块时才更新块头 SCN 和事务状态。
8. **异步落盘**：DBWn 择机把脏块写到数据文件；CKPT 推进检查点，更新控制文件和数据文件头。
9. **崩溃恢复**：实例异常终止后，SMON 先用 redo 前滚（包括 undo 块的变更），再回滚未提交事务，保证「已提交不丢、未提交不可见」。

**为什么 Oracle 提交快**：提交只要求 redo 落盘（顺序写），不要求数据块落盘（随机写）。这就是 WAL 思想：**用顺序写的日志换随机写的数据文件**。

### 3.4 redo、LGWR 与检查点【核心】

- **redo 记录的是「块怎么变」**，既有数据块的变更，也有 undo 块的变更，所以恢复时能「先前滚、再回滚」。
- **LGWR 的触发时机**：用户提交、redo log buffer 用到 1/3、积攒超过 1 MB、每 3 秒、DBWn 需要推进检查点时。前两条决定了提交延迟。
- **日志切换与检查点**：一个在线日志组写满就切换到下一组，触发检查点，CKPT 要求 DBWn 把脏块写到「该日志对应的位置之前」。日志组太少或太小会导致切换过于频繁，表现为 `log file switch (checkpoint incomplete)`，业务提交被拖慢。
- **增量检查点**：CKPT 维护一个检查点队列，持续小批量推进，避免一次性刷大量脏块造成 IO 抖动；`FAST_START_MTTR_TARGET` 用「期望恢复时间」反推检查点频率。
- **日志复用与多路**：在线日志至少两组，生产建议每组至少两个成员放在不同磁盘，防止单点损坏。日志文件不是越大越好，太大则崩溃恢复要前滚更多 redo。

### 3.5 归档模式与日志切换【重点】

- **NOARCHIVELOG**：日志被覆盖即丢失，只能做一致性关闭后的冷备，故障后最多恢复到上次备份点。
- **ARCHIVELOG**：ARCn 在切换后把日志归档，才能做**热备、RMAN 增量、任意时间点恢复、Data Guard**。
- 生产必须开归档，并且归档路径要有空间监控，否则报 **ORA-00257**，数据库直接挂起（所有 DML 都会阻塞）。
- `LOG_ARCHIVE_DEST_n` 可以同时归档到本地和远端（DG 的 redo 传输也走这里），`LOG_ARCHIVE_FORMAT` 决定文件名格式，跨平台/跨库恢复时会用到。

### 3.6 库内调度与并行执行【重点】

- **DBMS_SCHEDULER**（替代老的 DBMS_JOB）：Job、Program、Schedule、Window、Chain、Job Class 分离设计，支持失败重试、日志、依赖链。
- **资源管理器（Resource Manager）**：用 Consumer Group 限制 CPU、并行度、活跃会话数；多租户下 PDB 级也靠它做资源隔离。
- **并行执行**：并行查询/并行 DML 会启动 PX 从属进程，`DOP` 决定并行度；`PARALLEL_DEGREE_POLICY=AUTO` 让 Oracle 按代价自动决定。并行不是越大越快：并行度超过 IO 或 CPU 能力会互相抢资源，RAC 下还会放大 interconnect 流量。
- **边界**：库内调度适合「和数据强相关、逻辑轻」的批处理（日终汇总、分区维护）；复杂的任务编排、跨系统依赖放外部调度（XXL-JOB、Airflow），库内只做被调用方。

## 四、关键机制

### 4.1 一致性读与 SCN【核心】

SCN（System Change Number）是 Oracle 内部的逻辑时钟，在提交、检查点、日志切换等时刻推进。**一致性读（Consistent Read）**的流程：

1. 语句开始执行时记录一个 SCN（快照点）。
2. 读到某个块时比较块头 SCN：若块 SCN 小于等于快照 SCN，说明块是「旧的」，直接用。
3. 若块 SCN 更大，说明块被之后的事务改过，就从 undo 里找前镜像，**在 buffer 中构造一个 CR 块**（不改磁盘、不改原块）。
4. 用 CR 块返回结果，保证同一语句看到的是同一时间点的数据。

**三个结论**：

- Oracle **写不阻塞读、读不阻塞写**：读者靠 undo 看旧版本，写者只锁行。
- Read Committed 下一致性是**语句级**的：同一事务里两条 SELECT 可能看到不同结果（因为各自有快照点）。
- Serializable 下一致性是**事务级**的：整个事务共享事务开始时的 SCN，冲突时报 **ORA-08177**。Oracle 没有「读未提交」，因为它不需要靠锁来避免脏读。

### 4.2 ORA-01555 与延迟块清除【重点】

`ORA-01555: snapshot too old` 是一致性读的反面：需要的 undo 已经被覆盖，CR 块构造不出来。

**常见原因**：

1. **undo 表空间太小或 UNDO_RETENTION 太短**，长查询还没跑完，undo 就被新事务复用。
2. **fetch across commit**：游标打开后循环里频繁提交，后续 fetch 需要早期的 undo。
3. **延迟块清除**：提交时块没清理，很久以后第一次被读，需要查很早以前的 undo，如果 undo 已过期就报错。
4. 大批量更新/删除产生海量 undo，把表空间撑爆后自动复用。

**处理**：调大 undo 表空间、延长 `UNDO_RETENTION`、必要时 `RETENTION GUARANTEE`；减少循环内提交；把大查询拆小或改用批量抽取；对延迟块清除，可以在低峰期做一次全表扫描/`DBMS_STATS` 触发表扫描，把块的清理提前做掉。

### 4.3 事务与隔离级别【核心】

| 隔离级别 | Oracle 实现 | 现象 |
| --- | --- | --- |
| Read Committed（默认） | 语句级快照 + undo | 无脏读、无不可重复读之外的问题；同一事务两次查询可能不同 |
| Serializable | 事务级快照，检测到不能串行化就报 ORA-08177 | 强一致，但应用要能处理重试 |
| Read Only | 事务级只读快照 | 长时间报表用，不能 DML |
| （无 Read Uncommitted） | — | Oracle 不做脏读，不需要该级别 |

**事务的生命周期**：第一条 DML 或 `SELECT ... FOR UPDATE` 开始 → 分配 undo 段 → 修改数据并持锁 → `COMMIT`（redo 落盘、SCN 推进、锁释放）或 `ROLLBACK`（用 undo 回滚、释放锁）。DDL 是**隐式提交**，这也是「存储过程里不要随便 DDL」的原因。

### 4.4 锁：TM/TX、enqueue 与 latch【核心】

Oracle 的锁分两大类，面试必须分清：

| 类别 | 例子 | 特点 |
| --- | --- | --- |
| Enqueue（队列锁） | TM（表级意向锁）、TX（事务锁/行锁） | 可排队、可等待、记录在 `V$LOCK`，等待表现为 `enq: TX - ...` |
| Latch / Mutex | shared pool latch、library cache mutex、buffer cache 的 hash bucket latch | 内存结构的短锁，**不排队**，拿不到就自旋，表现为 `latch: ...`、`cursor: pin S wait on X` |

- **行锁在块里**：TX 锁并不为每一行分配锁结构，行锁信息写在数据块的 ITL 中，所以 Oracle 行锁几乎无内存开销，能支撑极高并发。
- **TM 锁**：DML 前对表加 RS/RX 意向锁，保护表结构不被并发 DDL 破坏；`ALTER TABLE` 要拿 X 锁，会被正在跑的 DML 挡住（`enq: TM - contention`），所以**DDL 要放在低峰或维护窗口**。
- **死锁**：两个会话互相等待对方持有的行锁，Oracle 约 3 秒内检测到，牺牲其中一个并报 **ORA-00060**，deadlock graph 写在 trace 文件里。预防手段：统一加锁顺序、缩短事务、确保走索引（避免锁住过多行）、队列消费用 `SKIP LOCKED`。
- **`SELECT ... FOR UPDATE`**：加行级排他锁；`NOWAIT` 立即失败，`WAIT n` 等 n 秒，`SKIP LOCKED` 跳过已锁行——这是实现「任务抢占」的标准写法。

### 4.5 索引类型与访问路径【核心】

| 类型 | 适用场景 | 注意点 |
| --- | --- | --- |
| B*Tree（普通/唯一） | 绝大多数 OLTP 等值与范围查询 | 叶子存 key + rowid；单列索引不存 NULL；可压缩、可反向 |
| Bitmap | 低基数（性别、状态）、数据仓库 | DML 并发差，位图段锁会让并发更新互相阻塞 |
| Function-Based | `UPPER(name)`、`TO_CHAR(dt,'YYYY')` 这类查询 | 查询表达式必须与索引表达式匹配；需要统计信息 |
| 反向键（Reverse Key） | 序列主键插入热点（RAC 尤甚） | 牺牲范围扫描能力 |
| 分区索引（Local/Global） | 大表按时间分区 | Local 维护简单；全局唯一约束只能建全局索引 |
| 域索引（Text/Spatial） | 全文、空间检索 | 独立选项，知道即可 |

**和 MySQL 的重要差异**：Oracle 二级索引叶子存的是 **rowid**（物理地址），不是主键；所以 Oracle 不存在「回表到聚簇索引」这一步，按 rowid 直接定位块。行迁移后 rowid 不变（原块留转发地址），但要多跳一次 IO。

**索引不被使用（≈ MySQL 的「索引失效」）**：

1. 对索引列做函数或运算（`WHERE UPPER(name)=...`、`WHERE id+1=10`），除非建函数索引。
2. 隐式类型转换（字符串列传数字、`NUMBER` 与 `VARCHAR2` 比较）。
3. `LIKE '%xx'` 前导通配。
4. 统计信息过期，CBO 估算选择性差而选择全表扫描。
5. 查询返回行占比高（比如超过 10%~20%），全表扫描更便宜。
6. 单列索引遇到 `IS NULL`（NULL 不入索引）；复合索引全 NULL 也不入。
7. 索引列上使用了 `!=`、`NOT IN` 等，CBO 估算成本高时放弃。
8. 索引聚簇因子（clustering factor）很差：索引顺序与表物理顺序严重不一致，回表成本高。

### 4.6 执行计划与连接方式【核心】

**常用查看方式**：

```sql
EXPLAIN PLAN FOR SELECT ...;
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);

-- 真实执行统计（需要 V$SQL 的 sql_id）
SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('&sql_id', NULL, 'ALLSTATS LAST'));
```

**访问路径**：全表扫描（TABLE ACCESS FULL）、索引唯一扫描（INDEX UNIQUE SCAN）、索引范围扫描（INDEX RANGE SCAN）、索引全扫描（INDEX FULL SCAN，按序读）、索引快速全扫描（INDEX FAST FULL SCAN，多块读但不保证有序）、索引跳跃扫描（INDEX SKIP SCAN，前导列基数低时可用）、ROWID 扫描。

**连接方式**：

| 方式 | 适用 | 风险 |
| --- | --- | --- |
| Nested Loops | 驱动表小、被驱动表有高效索引 | 驱动行数估算错误时会放大成百万次随机 IO |
| Hash Join | 大表与大表、等值连接 | 需要 PGA 哈希区，内存不足会溢出 temp |
| Sort Merge Join | 非等值或已有排序 | 排序成本高 |
| 星型转换 | 数据仓库星型模型 | OLTP 基本不用 |

**看计划要抓三件事**：估算行数（Cardinality）与实际行数（A-Rows）差多少、cost 花在哪一步、有没有隐式类型转换或全表扫描。**行数估算错**是计划劣化的第一原因，往下追就是统计信息问题。

### 4.7 存储程序：PL/SQL 与依赖管理【重点】

- **结构**：匿名块、存储过程、函数、包（规范 + 包体）、触发器。包里可以封装常量、类型、私有过程，是企业级 PL/SQL 的常见组织方式。
- **异常处理**：`NO_DATA_FOUND`、`TOO_MANY_ROWS`、`OTHERS`；`RAISE` 重新抛出，`SQLCODE/SQLERRM` 取错误信息。**不要用 `WHEN OTHERS THEN NULL` 吞异常**。
- **自治事务**：`PRAGMA AUTONOMOUS_TRANSACTION`，用于日志、审计——主事务回滚时日志仍要保留。但自治事务会独立提交，滥用会破坏事务边界。
- **包状态**：包的全局变量是**会话级**的，在连接池下同一会话复用会带来「变量残留」问题。
- **依赖与失效**：表结构变更后依赖对象变 `INVALID`，首次调用自动重编译；大范围 DDL 后建议 `utlrp.sql` 统一重编译。在线升级可用 **Edition-Based Redefinition**（版本化对象 + 切换 edition），实现不停机发布。
- **要不要用存储过程**：批量计算下推到库内能减少网络往返、利用并行；代价是版本管理难、测试难、占用数据库 CPU、迁移时绑定 Oracle。支付类系统一般把核心逻辑放应用层，库内只保留少量批处理与数据维护。

## 五、性能与容量

### 5.1 CBO 与统计信息【核心】

Oracle 从 9i 起默认用 **CBO（基于代价的优化器）**，代价由「估算行数 × 单次访问成本 + CPU 成本」算出，而估算行数完全依赖**统计信息**。

- **采集**：`DBMS_STATS.GATHER_TABLE_STATS`（带 `CASCADE=>TRUE` 连索引一起采）；默认有自动统计任务在维护窗口跑；分区表用 `INCREMENTAL=>TRUE` 只采变化分区。
- **关键统计**：表行数、块数、平均行长、列的不同值个数（NDV）、**直方图**、索引的**聚簇因子**。
- **直方图**：数据倾斜的列（如状态、地区）需要直方图，CBO 才能对不同的值给出不同行数估算；均匀分布建直方图反而浪费。
- **动态采样**：统计信息缺失或过旧时，优化器在硬解析时抽样，级别由 `OPTIMIZER_DYNAMIC_SAMPLING` 控制（默认自动）。能救急，但会增加硬解析开销。
- **计划稳定性**：`SQL Plan Baseline`（SPM）把验证过的好计划固定下来，新计划必须证明更优才启用；`SQL Profile` 用于修正估算偏差；老版本还有 stored outline。
- **12c 自适应计划**：运行时发现估算偏差过大，自动在 nested loops 与 hash join 之间切换。**19c 自动索引**：`DBMS_AUTO_INDEX` 在验证后自动创建/删除索引，默认关闭，要用需评估。

**排查顺序**：执行计划突变 → 先比对统计信息采集时间与数据增长 → 再看绑定变量窥探与直方图 → 最后才考虑 hint/基线。**先用 hint 不用统计信息，是典型的治标不治本**。

### 5.2 绑定变量、窥探与游标共享【核心】

- **好处**：减少硬解析、提高 Shared Pool 命中、天然防 SQL 注入。
- **绑定变量窥探（Bind Peeking）**：硬解析时优化器会「偷看」绑定变量的实际值来估算行数；如果列数据倾斜，第一次传入的值会决定整个游标的计划，后续传其他值就沿用这个可能很差的计划。
- **ACS（自适应游标共享）**：11g 起为同一 SQL 生成多个子游标，按不同的绑定值区间各自选计划，缓解窥探问题。
- **`CURSOR_SHARING=FORCE`**：把字面量自动替换成绑定变量，能救「拼 SQL 导致硬解析爆炸」的存量系统，但可能产生不适合的计划，属于过渡手段而不是长期方案。
- **分区裁剪的坑**：分区键用绑定变量时，**解析阶段无法裁剪分区**，只能运行时逐分区探测；分区表的查询尽量用字面量或 `DBMS_SQL` 绑定感知（12c）。
- **实践**：OLTP 全量使用预编译语句；倾斜列谨慎建直方图；关键 SQL 用 SPM 固定计划；监控 `V$SQL_SHARED_CURSOR` 看游标不共享的原因。

### 5.3 高级 SQL 能力【重点】

- **分析函数**：`ROW_NUMBER/RANK/DENSE_RANK`、`LAG/LEAD`、`SUM() OVER (PARTITION BY ... ORDER BY ...)`、`KEEP (DENSE_RANK FIRST/LAST)`。分组取 Top N、环比同比、累计求和都用它，比自连接高效得多。
- **`MERGE INTO`**：按条件做「有则更新、无则插入」，ETL 场景常用；注意并发下的唯一约束冲突。
- **`WITH` 子查询**：12c 起优化器可以内联或物化，递归 `WITH` 做树形/层级查询。
- **`PIVOT/UNPIVOT`**：行列转换；`MODEL` 做 spreadsheet 式计算；`MATCH_RECOGNIZE`（12c）做事件序列模式匹配。
- **分页**：12c 起支持 `OFFSET n ROWS FETCH NEXT m ROWS ONLY`；老写法是 `ROWNUM` 嵌套两层。深分页仍要靠「带上一页最大 key 的游标分页」而不是 `OFFSET`。
- **Flashback Query**：`AS OF TIMESTAMP/SCN` 直接查历史快照，排障时不用恢复库就能看到误删前的数据。

### 5.4 空间管理：ASSM、HWM 与在线整理【重点】

- **ASSM**：段内用三级位图（L1/L2/L3）管理空闲块，插入时自动找合适块，避免 freelist 争用。
- **PCTFREE**：块内为「更新导致行变长」预留的空间；预留太小会行迁移，太大浪费空间。
- **高水位线（HWM）**：插入只推进 HWM，删除不降低；全表扫描要读到 HWM，所以「删了 90% 数据表还是慢」。处理：`ALTER TABLE ... SHRINK SPACE`（需 ASSM 且开 row movement）、`ALTER TABLE ... MOVE ONLINE`（12c）、`TRUNCATE`（重置 HWM 但不可回滚）。
- **表空间**：本地管理表空间（LMT）是标准；`AUTOEXTEND` 要设上限，避免把磁盘写满；大文件表空间（bigfile）单文件可到数十 TB，适合超大数据文件；OMF 让 Oracle 自动命名管理文件。
- **分区**：Range（时间）、List（地区）、Hash（打散）、Interval（自动加分区）、Composite（Range + Hash/List）。`LOCAL` 索引随分区维护，`GLOBAL` 索引维护代价高但支持跨分区唯一约束；历史数据归档用 `EXCHANGE PARTITION` 秒级换出。
- **压缩**：Basic / OLTP / Advanced（HCC 需 Exadata）。OLTP 压缩适合读多写少、字段重复度高的表，代价是 DML 的 CPU 开销。

### 5.5 内存管理：ASMM、AMM 与 HugePages【重点】

- **ASMM**（`SGA_TARGET`）：自动在 Shared Pool、Buffer Cache 等之间调；PGA 仍独立由 `PGA_AGGREGATE_TARGET` 管。**生产最常见**。
- **AMM**（`MEMORY_TARGET`）：SGA + PGA 统一管理，依赖 `/dev/shm`，**与 HugePages 不能共存**，容易在容器/云主机上踩坑。
- **HugePages**：大页减少页表开销、避免 swap，是 Oracle 在 Linux 上的标准配置；要用 `USE_LARGE_PAGES=ONLY` 并预先分配足够大页。
- **Buffer Cache**：`V$DB_CACHE_ADVICE` 估算不同大小的收益；`KEEP` 池放热点小表，`RECYCLE` 池放一次性大表，防止全表扫描把缓存冲掉。
- **In-Memory Column Store**（12c）：`INMEMORY_SIZE`，按列压缩存内存，IM 扫描做过滤和聚合极快，适合报表/分析；需要 In-Memory 选项授权。
- **PGA 告警**：`V$PGASTAT` 的 `over allocation count`、`cache hit percentage`；连接数暴涨时先看 PGA 与进程数，必要时限制连接或上连接池。

### 5.6 性能优化方法论【重点】

1. **先量化**：用 AWR 看 DB Time 与 AAS（平均活跃会话），确认是「数据库忙」还是「应用在等」。
2. **定位层级**：Top 等待事件 → 对应 SQL（SQL ordered by Elapsed Time / CPU）→ 执行计划 → 对象统计。
3. **区分瓶颈类型**：IO 型（`db file sequential/scattered read`）、提交型（`log file sync`）、并发型（`enq: TX`、`buffer busy waits`）、解析型（`library cache`）、内存型（`free buffer waits`、PGA 溢出）。
4. **改的顺序**：SQL 与索引 → 统计信息 → 应用调用模式（N+1、批量）→ 参数 → 架构（分区、读写分离、RAC/ADG）。
5. **常见误区**：一上来调 `SGA_TARGET`、开大并行、加 hint；忽略绑定变量与连接池；把「AWR 里某条 SQL 排第一」当成根因，而不看它是否被高频调用。

## 六、可靠性与排障

### 6.1 等待事件体系【核心】

**DB Time = DB CPU + 非空闲等待时间**，AAS = DB Time / 采样时长。等待事件按类别分：User I/O、System I/O、Concurrency、Application、Commit、Network、Configuration、Administrative、Cluster、Idle。

| 等待事件 | 含义 | 常见原因 |
| --- | --- | --- |
| `db file sequential read` | 单块读（索引、回表） | 索引选择性差、回表多、缓存不足 |
| `db file scattered read` | 多块读（全表、快速全扫） | 缺索引、大表扫描、并行度不合理 |
| `direct path read/write` | 绕过 buffer cache 的 IO | 并行查询、直接路径加载、temp 排序 |
| `log file sync` | 用户提交等 LGWR 刷盘 | 提交过于频繁、日志盘慢、日志组太小 |
| `log file parallel write` | LGWR 自身写日志 | 日志盘 IO 瓶颈 |
| `db file parallel write` | DBWn 写脏块 | 检查点过密、存储慢 |
| `buffer busy waits` | 多个会话抢同一个块 | 热块、ITL 不足、位图索引并发 |
| `free buffer waits` | 找不到干净 buffer | DBWn 写不过来、buffer cache 太小 |
| `enq: TX - row lock contention` | 行锁等待 | 长事务、更新顺序不一致 |
| `enq: TX - allocate ITL entry` | 块内 ITL 槽不够 | 高并发写同表，需调 INITRANS |
| `enq: TM - contention` | 表级锁等待 | DDL 与 DML 冲突、外键未建索引 |
| `latch: shared pool` / `cursor: pin S wait on X` | 解析争用 | 硬解析多、绑定变量缺失 |
| `gc buffer busy acquire` 等 `gc` 系列 | RAC 全局缓存争用 | 热点块、跨实例访问频繁 |

**Idle 等待不计入 DB Time**，比如 `SQL*Net message from client`（等客户端发请求），别把它当性能问题。

### 6.2 AWR / ASH / ADDM 怎么用【核心】

- **AWR**：MMON 按小时采集快照（默认保留 8 天），`@?/rdbms/admin/awrrpt.sql` 生成报告。重点看 **DB Time、Top 5 Timed Foreground Events、SQL ordered by Elapsed Time/CPU、Instance Efficiency、Segment Statistics**。
- **ASH**：每秒采样一次活跃会话，实时在 `V$ACTIVE_SESSION_HISTORY`，历史在 `DBA_HIST_ACTIVE_SESS_HISTORY`。适合回答「刚才那 10 分钟到底谁在等什么」，比 AWR 的分钟级粒度更细；`ashrpt.sql` 出报告。
- **ADDM**：自动分析两个快照之间的瓶颈并给建议，适合快速定位「最值得优化的那一件事」。
- **Real-Time SQL Monitoring**：执行超过 5 秒（或并行）的 SQL 自动监控，`DBMS_SQLTUNE.REPORT_SQL_MONITOR` 看每一步的实际行数与耗时，是排查单条慢 SQL 的利器。
- **成本边界**：AWR/ASH/ADDM/SQL Monitor 属于 Diagnostics Pack / Tuning Pack，**要单独授权**；没有授权时用 `V$SESSION`、`V$SQL`、`V$SYSTEM_EVENT` 定时采样，或 Statspack（免费）。

### 6.3 阻塞与锁等待排查【核心】

**排查链路**：谁在等 → 等什么 → 谁阻塞了它 → 阻塞者在跑什么 → 是不是「事务开了没提交」。

```sql
-- 阻塞树：找根阻塞者
SELECT s.sid, s.serial#, s.username, s.status, s.event, s.seconds_in_wait,
       s.blocking_session, s.sql_id, s.prev_sql_id
FROM v$session s
WHERE s.blocking_session IS NOT NULL;

-- 锁明细
SELECT sid, type, id1, id2, lmode, request, ctime, block
FROM v$lock
WHERE block = 1 OR request > 0;

-- 被阻塞者与阻塞者
SELECT * FROM dba_waiters;
SELECT * FROM dba_blockers;
```

**处理**：

- 优先让业务方提交或回滚，锁自然释放；确认是废弃会话再 kill。
- `ALTER SYSTEM KILL SESSION 'sid,serial#' IMMEDIATE`：会话标记 KILLED，由 SMON 回滚事务；**大事务回滚可能要很久**，期间资源仍被占用。
- 操作系统层 `kill -9` 前台进程会触发 PMON 清理；对后台核心进程 `kill -9` 可能导致实例崩溃，不要随便做。
- 预防：短事务、统一加锁顺序、外键列建索引（否则父表 DML 会锁全表子表相关行）、队列消费用 `SKIP LOCKED`、DDL 放维护窗口。

### 6.4 备份与恢复：RMAN【核心】

- **基本概念**：target 数据库、可选 catalog 库、channel（并行通道）、backup set / piece、image copy（可直接用文件系统恢复）、retention policy（保留策略）、FRA（快速恢复区）。
- **备份类型**：全备、增量 level 0（等价全备基线）、level 1 差异（自上次增量）、level 1 累积（自上次 level 0）；归档日志备份；控制文件自动备份；`BLOCK CHANGE TRACKING` 让增量备份只读变化的块。
- **常用命令**：

```sql
RMAN> BACKUP INCREMENTAL LEVEL 0 DATABASE PLUS ARCHIVELOG;
RMAN> BACKUP INCREMENTAL LEVEL 1 DATABASE PLUS ARCHIVELOG;
RMAN> RESTORE DATABASE PREVIEW;              -- 恢复预演
RMAN> RESTORE DATABASE; RECOVER DATABASE;    -- 完全恢复
RMAN> RESTORE DATABASE UNTIL TIME "TO_DATE('2026-11-20 10:00:00','YYYY-MM-DD HH24:MI:SS')";
RMAN> RECOVER DATABASE UNTIL TIME "TO_DATE('2026-11-20 10:00:00','YYYY-MM-DD HH24:MI:SS')";
```

- **不完全恢复**要求归档日志连续且完整；恢复后必须 `ALTER DATABASE OPEN RESETLOGS`，会重置日志序列，务必先确认。
- **NOARCHIVELOG 只能恢复到备份点**，所以生产必须开归档。
- **逻辑备份**：Data Pump（`expdp/impdp`），支持并行、`REMAP_SCHEMA/REMAP_TABLESPACE`、`FLASHBACK_SCN` 一致性导出；适合迁移、单表恢复，不适合整库容灾。
- **验证**：`RESTORE VALIDATE` / `BACKUP VALIDATE` 检查备份可恢复性；备份不验证等于没备份。

### 6.5 闪回家族【重点】

| 能力 | 依赖 | 适用 |
| --- | --- | --- |
| Flashback Query（`AS OF`） | undo | 查误删/误改前的数据 |
| Flashback Version Query | undo | 看一行在时间区间内的所有版本及事务 |
| Flashback Table | undo + `ROW MOVEMENT` | 把整表回退到某时间点（不涉及其他表） |
| Flashback Drop | Recyclebin | 恢复 `DROP TABLE`，`FLASHBACK TABLE t TO BEFORE DROP` |
| Flashback Database | flashback logs（需开启） | 整库回退到过去某时刻，需在 mount 状态执行 |
| Restore Point | normal / guaranteed | 打标记，配合闪回或恢复使用 |

**边界**：闪回依赖 undo 保留时间或 flashback log 空间，**不能替代备份**——介质损坏、文件丢失只能靠 RMAN；`FLASHBACK DATABASE` 之后还要用归档日志前滚到目标时间点。

### 6.6 常见 ORA 错误速查【重点】

| 错误 | 含义 | 首查方向 |
| --- | --- | --- |
| ORA-01555 | snapshot too old | undo 大小/保留时间、循环内提交、延迟块清除 |
| ORA-00060 | 检测到死锁 | trace 里的 deadlock graph、加锁顺序 |
| ORA-00257 | 归档空间满 | FRA / 归档目录空间、删除过期归档 |
| ORA-04031 | 共享池无法分配内存 | 硬解析过多、Shared Pool 碎片、调大或刷新 |
| ORA-01652 | 临时段无法扩展 | temp 表空间、排序/哈希溢出、并行度 |
| ORA-04036 | PGA 超限 | 连接数、排序并发、`PGA_AGGREGATE_LIMIT` |
| ORA-08177 | 无法串行化访问 | Serializable 隔离级别下的冲突，应用需重试 |
| ORA-30036 | undo 表空间无法扩展 | undo 空间、`RETENTION GUARANTEE`、大事务拆分 |
| ORA-00054 | 资源忙 | 有未提交事务或 DDL 持锁，加 `NOWAIT` 或等窗口 |
| ORA-12514/12541 | 监听无服务/连不上 | 监听状态、服务名、`tnsnames.ora` |

**排障第一入口是告警日志**：`$DIAGNOSTIC_DEST/diag/rdbms/<db>/<instance>/trace/alert_<instance>.log`，用 ADRCI 查看与打包 incident。

## 七、生态与演进

### 7.1 RAC：共享存储多实例【核心】

- **架构**：多台机器各跑一个实例，共享同一套存储（通常 ASM），通过私网 interconnect 互联；由 Grid Infrastructure 的 Clusterware 管理。
- **Cache Fusion**：块在实例间通过 interconnect 传递，GCS 负责块访问、GES 负责全局队列（锁）；等待表现为 `gc current block 2-way/3-way`、`gc cr block ...`、`gc buffer busy`。
- **关键组件**：SCAN（单客户端入口）、VIP（节点故障时漂移）、OCR（集群配置）、Voting Disk（仲裁）。
- **高可用**：节点故障时，存活实例接管并回滚失败实例的未提交事务（`SMON` 做实例恢复），客户端通过 TAF/FAN 或 Application Continuity（12c，可重放事务）重连。
- **扩展的边界**：RAC 能横向扩展连接与部分读写能力，但**热点块、序列、索引右端争用**会放大；常见优化是反向键索引、序列 `CACHE`/`NOORDER`、分区打散。RAC 不是分库分表，解决的是单库高可用，不是海量数据水平拆分。
- **跨机房**：Extended RAC 对 interconnect 延迟极敏感，通常用 Data Guard 做异地，而不是拉长 RAC。

### 7.2 Data Guard：异地容灾【核心】

| 类型 | 原理 | 特点 |
| --- | --- | --- |
| 物理 standby | 块级 redo 应用 | 与主库完全一致，可做 Active Data Guard 只读查询 |
| 逻辑 standby | SQL 应用（LogMiner 解析） | 可读写、可做滚动升级，但限制多、易冲突 |
| Snapshot standby | 临时打开读写 | 用于演练/测试，之后可回退 |

- **保护模式**：最大保护（SYNC，零数据丢失，备库异常会 hang 主库）、最高可用（SYNC，备库不可用时自动降级为异步）、最大性能（ASYNC，主库不等待）。
- **redo 传输**：SYNC / ASYNC / FASTSYNC；12c 的 **Far Sync** 实例放在近端，先把 redo 落地再异步传远端，兼顾低延迟与低丢失。
- **切换**：switchover 是计划内角色互换，failover 是灾难切换；DG Broker 统一管理；**FSFO**（Fast-Start Failover）+ observer 可自动切换。
- **Active Data Guard**：备库实时应用的同时开放只读，分担报表/查询，需要单独授权。
- **RAC 与 DG 的分工**：RAC 解决同机房高可用与扩展，DG 解决异地容灾，两者常组合成「本地 RAC + 异地 DG」。

### 7.3 多租户 CDB/PDB【重点】

- **结构**：CDB（容器数据库）包含 `CDB$ROOT`、`PDB$SEED` 和多个 PDB；PDB 是可插拔的业务库，共享同一套实例资源。
- **用户**：公共用户（`C##` 前缀）在 CDB 级创建，本地用户在 PDB 内创建；权限、表空间、对象都按容器隔离（`CON_ID`）。
- **常用操作**：`CREATE PLUGGABLE DATABASE`、本地克隆 / 远程克隆 / **热克隆**（12.2+，源库在线）、`UNPLUG/PLUG`、在线 `RELOCATE`、PDB 快照轮播（18c，用于快速回滚）。
- **资源与隔离**：PDB 级 CPU/IO 份额、`MAX_PDBs`、Lockdown Profile 限制 PDB 的能力；但 PDB 共享后台进程和 SGA，**不是完全隔离**，一个 PDB 的异常仍可能影响整个 CDB。
- **运维**：补丁以 CDB 为单位（`datapatch`），升级以 CDB 为单位；单实例下少量用户 PDB 的授权例外以官方 Licensing Guide 为准。

### 7.4 数据同步：GoldenGate / LogMiner / Data Pump【重点】

- **GoldenGate**：从 redo/归档里抽取变更（Extract）→ 写 trail 文件 → 投递（Pump）→ 目标端应用（Replicat）。支持异构（Oracle → MySQL/Kafka）、双向复制（需冲突处理）、DDL 复制、集成模式（Integrated Extract/Replicat 利用 LogMiner）。
- **LogMiner**：用 SQL 解析 redo/归档，`V$LOGMNR_CONTENTS` 看变更；需要开启补充日志（`SUPPLEMENTAL LOG DATA`）。很多 CDC 工具（Debezium Oracle Connector、Flink CDC）底层就是 LogMiner。
- **常见组合**：Data Pump 做全量初始化 + GoldenGate/CDC 做增量，切换时校验一致性再切流。
- **其他**：物化视图（Materialized View）定时刷新做报表汇总（fast refresh 需要 MV Log）；DB Link 做跨库查询；外部表/SQL*Loader 做文件导入；Streams 已弃用，不要新用。

### 7.5 版本演进与云上形态【了解】

- **版本线**：11g（RAC/ASM 成熟）→ 12c（多租户、In-Memory、ADG、自适应计划）→ 18c/19c（长期支持、自动索引、PDB 快照轮播）→ 21c → **23ai**（AI Vector Search、JSON Relational Duality、更多 schema 级权限）。
- **云上形态**：自治数据库（ADB）自动打补丁、自动调优、自动扩缩；Exadata 提供智能扫描（把过滤和列投影下推到存储节点）和 HCC 压缩。
- **演进趋势**：新项目更多选 MySQL/PG/云原生数据库，Oracle 的增量集中在存量核心系统与混合负载（HTAP）场景；23ai 的向量检索是想把 AI 应用也留在库内。
- **迁移注意点**：PL/SQL 改写、hint 与执行计划重写、序列/分区/LOB 语义、字符集与排序规则、事务隔离差异、`ROWNUM` 与 `LIMIT` 的改写、隐式类型转换行为差异。迁移的难点从来不是「语法」，而是「行为一致性验证」。

## 八、高频面试题与自测清单

### 8.1 高频面试题

| # | 题目 | 答案要点 | 章节 |
| --- | --- | --- | --- |
| 1 | 实例和数据库有什么区别？ | 实例 = SGA + 后台进程；数据库 = 数据文件 + redo + 控制文件；RAC 多实例对一库 | 二 |
| 2 | SGA 由哪些部分组成？ | Shared Pool、Buffer Cache、Redo Log Buffer、Large Pool、Java/Streams Pool、Fixed SGA | 二 |
| 3 | PGA 和 SGA 的区别？ | 共享 vs 私有；随连接增长；排序/哈希溢出到 temp；PGA_AGGREGATE_LIMIT | 二 |
| 4 | 逻辑存储结构是什么？ | 表空间 → 段 → 区 → 块；ASSM；undo/temp 表空间；行迁移 | 二 |
| 5 | 启动的三个阶段？ | NOMOUNT 读参数 → MOUNT 读控制文件 → OPEN 打开数据文件；SMON 做实例恢复 | 三 |
| 6 | 一条 SQL 的解析过程？ | 语法 → 语义/权限 → Library Cache 查找 → 软解析或硬解析 → 执行 | 三 |
| 7 | 硬解析为什么慢？怎么减少？ | CPU + latch/mutex 争用；绑定变量、游标缓存、预编译 | 三 |
| 8 | 一条 UPDATE 的完整流程？ | 分配 undo → 读块 → 占 ITL → 写 undo/redo → 提交等 LGWR → DBWn 异步写 | 三 |
| 9 | redo 和 undo 分别解决什么？ | redo 保证已提交不丢（前滚）；undo 支持回滚与一致性读 | 三 |
| 10 | COMMIT 到底做了什么？ | 写 commit 记录、推进 SCN、LGWR 刷 redo、释放锁；数据块不立即写、不清理 | 三 |
| 11 | 检查点和 LGWR/DBWn 什么关系？ | 日志切换触发检查点；CKPT 推进、DBWn 写脏块；增量检查点控制恢复时间 | 三 |
| 12 | 归档模式有什么用？ | 热备、任意时间点恢复、Data Guard；不开归档只能恢复到备份点 | 三 |
| 13 | 一致性读怎么实现？ | SCN 快照 + undo 构造 CR 块；写不阻塞读；RC 语句级、Serializable 事务级 | 四 |
| 14 | ORA-01555 的原因和处理？ | undo 太小/保留短、fetch across commit、延迟块清除；调大 undo、减少提交 | 四 |
| 15 | Oracle 的隔离级别有哪些？ | RC（默认）、Serializable、Read Only；没有脏读 | 四 |
| 16 | 行锁怎么实现？TM 和 TX 区别？ | 行锁在块 ITL 中；TX 事务锁、TM 表级意向锁；enqueue 与 latch 区别 | 四 |
| 17 | 死锁怎么发现和避免？ | ORA-00060 + deadlock graph；统一加锁顺序、短事务、走索引、SKIP LOCKED | 四 |
| 18 | B*Tree 和 Bitmap 索引怎么选？ | OLTP 用 B*Tree；低基数、DSS 用 Bitmap，但 DML 并发差 | 四 |
| 19 | 索引为什么会不被使用？ | 函数/运算、隐式转换、前导通配、统计信息过期、选择性差、NULL、聚簇因子差 | 四 |
| 20 | 执行计划怎么看？ | DBMS_XPLAN、A-Rows 与估算行数对比、访问路径、连接方式、cost 分布 | 四 |
| 21 | CBO 的计划为什么会突变？ | 统计信息变化、直方图、绑定变量窥探、数据增长；用 SPM 固定计划 | 五 |
| 22 | 绑定变量有什么坑？ | 窥探导致计划只适合首次值、分区裁剪失效；用 ACS/SPM 兜底 | 五 |
| 23 | AWR 和 ASH 的区别？ | AWR 小时级快照做趋势与 Top SQL；ASH 秒级采样定位瞬时瓶颈 | 六 |
| 24 | 线上会话被阻塞怎么排查？ | v$session.blocking_session → v$lock → 阻塞者的 SQL → 是否长事务 | 六 |
| 25 | RMAN 的备份策略怎么设计？ | 全备 + 增量 + 归档、保留策略、FRA、定期 RESTORE VALIDATE、异地副本 | 六 |
| 26 | 闪回能替代备份吗？ | 不能；依赖 undo/flashback log，介质损坏只能靠 RMAN | 六 |
| 27 | RAC 的 Cache Fusion 是什么？ | 块通过 interconnect 在实例间传递；gc 等待；热点块争用与优化 | 七 |
| 28 | DG 的三种保护模式？ | 最大保护、最高可用、最大性能；SYNC/ASYNC 与零丢失的取舍 | 七 |
| 29 | CDB 和 PDB 是什么？ | 多租户架构、公共/本地用户、资源隔离、以 CDB 为单位打补丁 | 七 |
| 30 | GoldenGate 的同步原理？ | 抽取 redo → trail → 投递 → 应用；与 LogMiner/CDC 的关系 | 七 |

**加分题**：为什么 Oracle 提交只需要 redo 落盘？延迟块清除为什么会引发 ORA-01555？`db file sequential read` 一定是索引读吗？为什么外键列不建索引会导致锁表？RAC 下序列为什么要 `NOORDER`？Data Pump 导出期间数据一致性怎么保证？`CURSOR_SHARING=FORCE` 有什么风险？19c 自动索引该不该开？

### 8.2 动手实验

- [ ] 用 Docker 起一个 Oracle Free（23ai）或 19c XE，执行 `startup` 三阶段并观察告警日志
- [ ] 用 `V$SGA`、`V$SGASTAT`、`V$PGASTAT` 观察内存分布，调整 `SGA_TARGET` 看变化
- [ ] 造一张 100 万行的表，对比全表扫描与索引范围扫描的 `db file sequential/scattered read`
- [ ] 开两个会话复现行锁等待，用 `v$session.blocking_session` 和 `dba_waiters` 找阻塞者
- [ ] 制造一次死锁，从 trace 里读懂 deadlock graph
- [ ] 用 `EXPLAIN PLAN` + `DBMS_XPLAN.DISPLAY_CURSOR(...,'ALLSTATS LAST')` 对比估算行数与实际行数
- [ ] 关闭绑定变量改用字面量，观察 `V$SQL` 里子游标数量和 `library cache` 等待变化
- [ ] 故意把 undo 表空间设小，跑长查询复现 ORA-01555
- [ ] 用 RMAN 做一次全备 + 增量 + 时间点恢复（在测试库）
- [ ] 用 `FLASHBACK TABLE ... TO BEFORE DROP` 恢复误删的表，再试 `FLASHBACK DATABASE`
- [ ] 生成一份 AWR 报告，找出 Top 5 等待事件和 Top SQL，并解释每一条的含义

### 8.3 自测清单

- [ ] 能不看书画出「实例 + 数据库」的文件与内存对应图
- [ ] 能讲清 SGA 各组件的作用和典型争用现象
- [ ] 能完整讲出 UPDATE 从分配到提交的 9 个步骤
- [ ] 能解释「提交只需要 redo 落盘」以及检查点如何控制恢复时间
- [ ] 能用 SCN 和 undo 讲清一致性读，并说明 RC 与 Serializable 的差别
- [ ] 能说出 ORA-01555 的至少 3 个原因和对应处理
- [ ] 能分清 enqueue 与 latch，讲清 TM/TX 锁和行锁存在哪里
- [ ] 能根据等待事件判断瓶颈属于 IO、提交、并发、解析还是内存
- [ ] 能独立完成一次阻塞排查并给出处理方案
- [ ] 能设计一套「全备 + 增量 + 归档 + 异地」的备份策略并说明恢复步骤
- [ ] 能讲清 RAC 与 Data Guard 的定位差异，以及 CDB/PDB 的隔离边界
