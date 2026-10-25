# Elasticsearch 核心知识点学习文档

Oct 25, 2026 · @meteorzhong

## 导读

本文展开《知识图谱.html》「数据库与存储」分类中的 Elasticsearch 节点（cat=db，图谱级别：重点），覆盖倒排索引、写入与近实时、分片副本与集群、DSL 检索与相关性、聚合与深分页、数据同步与运维六块内容。标注沿用图谱：**【核心】** 面试必问；**【重点】** 常被追问；**【了解】** 知道概念即可。默认版本为 Elasticsearch 7.x（以 7.17 为主，涉及 8.x 差异会单独注明），分词器默认提及 IK。

| 图谱节点 | 级别 | 本文章节 |
| --- | --- | --- |
| Elasticsearch（elasticsearch） | 重点 | 倒排索引、写入与近实时、分片副本与集群、检索与相关性、聚合与深分页、数据同步与运维 |

**复习顺序**：倒排索引与分词 → 写入链路与 refresh/flush → 分片副本与路由 → Query DSL 与打分 → 聚合与深分页 → MySQL 同步与运维。面试主线通常是「ES 为什么快、为什么近实时、数据怎么和 MySQL 对上」，能把倒排索引、写入流程、分片路由、同步方案四条讲顺即可。

## 一、核心概念与倒排索引

### 1.1 节点、集群、索引与文档【重点】

| 概念 | 含义 | 备注 |
| --- | --- | --- |
| 节点 Node | 一个 ES 进程实例，存储数据并参与索引与搜索 | 通过 `node.roles` 决定承担哪些角色 |
| 集群 Cluster | 多个节点组成的整体，共享数据、对外提供联合检索 | 由 `cluster.name` 标识 |
| 索引 Index | 逻辑上的文档集合，由若干主分片和副本组成 | 相当于关系库中的「表」 |
| 文档 Document | 可搜索的最小信息单元，JSON 格式 | 相当于「行」，有 `_id`、`_source` |
| 字段 Field | 文档中的一个键值 | 类型由 Mapping 定义 |
| 分片 Shard | 索引的物理切分，每个分片是一个独立的 Lucene 索引 | 主分片数创建后不可直接改 |

**Type 已移除**：5.x 之前一个索引可有多个 type，6.x 限制为一个，**7.0 正式移除 type**（写入用固定的 `_doc` 占位，查询不再需要指定），8.x 彻底删除相关 API。今天只需回答「type 是早期的逻辑分类，已废弃，一个索引只存一类文档」。

### 1.2 倒排索引的结构【核心】

正排索引是「文档 ID → 文档内容」，倒排索引反过来，是「词项 term → 包含它的文档列表」，这是全文检索能做快的根本原因。

| term | posting list |
| --- | --- |
| Java | 1, 2 |
| 面试 | 1 |
| 并发 | 2 |

一个倒排索引由三部分组成：

| 组成 | 内容 | 作用 |
| --- | --- | --- |
| 词项字典 Term Dictionary | 所有词项，按字典序排列 | 存放词项及其指向倒排列表的指针 |
| 词项索引 Term Index | 在字典之上建的前缀索引，用 FST 实现 | 体积小、常驻内存，快速定位词项在字典中的块 |
| 倒排列表 Posting List | 文档 ID、词频 TF、位置 position、偏移 offset | TF 用于打分，位置用于短语查询，偏移用于高亮 |

关键优化要能点出来：

- **FST（有限状态转换器）**：共享词项的前缀与后缀，把庞大的词项字典压缩成很小的词项索引放在内存，先查 FST 定位到磁盘上的块，再顺序扫描块内的词项。
- **Frame of Reference**：倒排列表里的文档 ID 有序，先做差值再按块用最少的位宽压缩，显著降低磁盘占用。
- **跳表**：多个倒排列表求交集时快速跳过不匹配的文档。
- **Roaring Bitmap**：filter 结果的缓存位图，压缩率高、交并运算快。
- **段不可变**：倒排索引按段（segment）生成，段一旦写出就不再修改，因此无需加锁、便于缓存；更新是「标记删除 + 写新文档」，删除靠 `.del` 文件标记，最终由段合并清理。
- **doc_values**：与倒排索引配套的列式正排存储，用于排序与聚合；`_source` 保存原始文档，用于取回与重建。

是否记录 TF、位置、偏移由 `index_options` 控制：`docs`（只有文档 ID）、`freqs`（加词频）、`positions`（加位置，text 默认）、`offsets`（加偏移）。关掉不需要的部分能省磁盘、提写入速度。

### 1.3 分词与分析器【核心】

分析（analysis）把一段文本转成词项序列，**只作用于 text 字段**，且在索引时与搜索时各执行一次。两边分析器不一致是「明明有数据却搜不到」的最常见原因。

分析器由三部分组成，按顺序执行：

| 组件 | 数量 | 作用 | 常见实现 |
| --- | --- | --- | --- |
| 字符过滤器 Character Filter | 0 到多个 | 分词前处理原始字符流 | `html_strip`、`mapping`、`pattern_replace` |
| 分词器 Tokenizer | 有且仅有 1 个 | 切分为 token，并记录位置与偏移 | `standard`、`whitespace`、`keyword`、`pattern`、`ngram`、`ik_smart`、`ik_max_word` |
| 词元过滤器 Token Filter | 0 到多个 | 对 token 增删改 | `lowercase`、`stop`、`stemmer`、`synonym`、`asciifolding` |

要点：

- 默认 `standard` 分析器对英文按词切分并转小写，**对中文切成单字**，中文场景必须换 IK 等中文分词器。
- `ik_max_word` 细粒度、召回高，适合索引时；`ik_smart` 粗粒度、准确率高，适合搜索时。常见做法是 `analyzer: ik_max_word` + `search_analyzer: ik_smart`。
- 同义词一般放在搜索侧的分析器中，便于热更新而不必重建索引。
- 调试用 `_analyze` API 直接看分词结果，例如 `GET /index/_analyze {"analyzer":"ik_smart","text":"Java 面试"}`。
- 自定义分析器写在索引 settings 的 `analysis` 段，组合上面三类组件。

### 1.4 Mapping 设计【重点】

Mapping 相当于表结构，定义字段类型与索引方式，**直接决定字段能不能搜索、能不能排序聚合**。

| 类型 | 是否分词 | 典型用途 | 注意 |
| --- | --- | --- | --- |
| `text` | 是 | 全文检索、模糊匹配 | 默认不能排序聚合，需 `keyword` 子字段 |
| `keyword` | 否 | 精确匹配、排序、聚合、过滤 | `ignore_above` 限制超长值 |
| 数值 / `date` / `boolean` | 否 | 范围查询、排序、聚合 | 用 `scaled_float` 存金额避免精度问题 |
| `object` | 否 | 普通嵌套 JSON | 对象数组会被扁平化，丢字段关联 |
| `nested` | 否 | 对象数组需独立查询 | 每个嵌套对象是独立 Lucene 文档，开销大 |
| `join` | 否 | 父子文档 | 父子必须路由到同一分片 |

字段级关键参数：

| 参数 | 说明 |
| --- | --- |
| `index: false` | 不建倒排索引，不可搜索，但仍在 `_source` 中，可省磁盘与写入开销 |
| `doc_values: false` | 关闭列式存储，字段不可排序聚合，节省磁盘 |
| `norms: false` | 关闭长度归一化因子，不需要打分时关闭 |
| `fields` | 多字段，如一个字段同时提供 text 与 keyword 两种视图 |
| `copy_to` | 把多个字段拼到一个字段，简化查询 |
| `analyzer` / `search_analyzer` | 分别指定索引时与搜索时的分析器 |
| `dynamic` | `true` 自动推断、`false` 忽略新字段、`strict` 报错、`runtime` 作为运行时字段 |
| `dynamic_templates` | 按规则定制动态映射，避免字段类型误判 |

设计原则：

- 生产环境**显式建 Mapping**，不要依赖动态映射；动态映射可能把数字识别成 text，也会因用户随意加字段导致字段爆炸。
- `index.mapping.total_fields.limit` 默认 1000，防止映射膨胀。
- **已有字段的类型不能修改**，只能新建索引 → `reindex` 迁数据 → 别名切换；可以新增字段，少数参数可改。

```json
PUT /article
{
  "settings": { "number_of_shards": 3, "number_of_replicas": 1 },
  "mappings": {
    "properties": {
      "title":   { "type": "text", "analyzer": "ik_max_word", "search_analyzer": "ik_smart",
                   "fields": { "raw": { "type": "keyword" } } },
      "author":  { "type": "keyword" },
      "content": { "type": "text", "analyzer": "ik_max_word", "index_options": "positions" },
      "views":   { "type": "long" },
      "tags":    { "type": "keyword", "index": false },
      "created": { "type": "date" }
    }
  }
}
```

### 1.5 近实时（NRT）的含义【重点】

写入的文档不是立刻可搜，默认要等一次 **refresh（约 1 秒）** 把内存 buffer 生成新段后才对搜索可见，所以叫「近实时」而不是实时。设计取舍是：每次写入都立即生成段会让段数量爆炸、合并压力巨大；用 1 秒的批量刷写换取吞吐。需要写后立即可见可以 `refresh=wait_for`（等下一次 refresh 返回）或 `refresh=true`（强制刷新，开销大，不要频繁用）。

## 二、写入与近实时

### 2.1 写入链路【重点】

| 步骤 | 执行者 | 动作 |
| --- | --- | --- |
| 1 | 协调节点 | 根据 `_routing` 计算目标分片，把请求转发给该分片的主分片 |
| 2 | 主分片 | 校验结构、执行写入：写 Lucene（buffer + translog），生成 `_seq_no` |
| 3 | 主分片 | 并行把操作转发给 in-sync 副本集合中的所有副本 |
| 4 | 副本 | 写入成功后应答主分片 |
| 5 | 主分片 | 所有副本应答后向协调节点返回成功 |
| 6 | 协调节点 | 汇总结果返回客户端 |

副本写失败会被移出 in-sync 集合并标记为 stale，之后重建，保证「在集合内的副本与主分片一致」。写入前可用 `wait_for_active_shards` 要求活跃分片数（默认 1，`all` 表示所有副本在线才写），这是可用性与一致性的取舍开关。

### 2.2 refresh、flush 与 translog【重点】

| 机制 | 触发 | 做什么 | 是否持久化 | 是否可搜 |
| --- | --- | --- | --- | --- |
| 内存 buffer | 每次写入 | 文档先进 indexing buffer | 否 | 否 |
| translog | 每次写入 | 追加操作日志，兜底未落盘的数据 | 是（按 durability 决定 fsync 时机） | 否 |
| refresh | 默认每 1 秒 | buffer 生成新 segment 并打开 reader | 否（只在文件系统缓存） | 是 |
| flush | translog 达阈值等 | Lucene commit、fsync 段文件、清空 translog | 是 | 是 |

- `index.refresh_interval` 默认 `1s`，全量导入时设为 `-1` 关闭，导入完再恢复。
- **translog 的作用是崩溃恢复**：refresh 只让数据可搜，不保证落盘，进程崩溃后靠 translog 重放恢复。
- `index.translog.durability` 默认 `request`，每次写请求都 fsync，最安全；改为 `async` 后按 `sync_interval`（默认 5 秒）刷盘，吞吐更高但可能丢这几秒的数据。
- `index.translog.flush_threshold_size` 默认 512MB，达到阈值触发 flush。
- 一句话总结：**refresh 管「可见」，flush 管「落盘」，translog 管「不丢」**。

### 2.3 Segment 与段合并【了解】

段是不可变的 Lucene 索引文件。不可变带来无锁读、文件缓存友好、易并发，代价是更新删除只能「标记 + 重写」。后台 merge 把多个小段合并成大段，同时物理删除被标记的文档。

- 段越少，查询时打开的文件越少、性能越好；但合并本身消耗大量磁盘 I/O 与 CPU。
- 对**只读索引**做 `_forcemerge?max_num_segments=1` 能显著降低段数和文件句柄，**不要在正在写入的索引上执行**。
- 写入高峰时 refresh 间隔太小会持续产生小段，合并跟不上，表现为写入变慢、磁盘 I/O 升高。

### 2.4 bulk 批量写入与写入优化【重点】

单条写入的网络往返与解析开销占比很高，生产必须用 `_bulk`。单批建议 5MB 到 15MB（用压测找最佳值），多线程并发发送，遇到 429（写线程池队列满）做指数退避重试，不要盲目加大并发。

| 优化项 | 做法 | 代价 |
| --- | --- | --- |
| 批量 | `_bulk`，单批 5–15MB，多线程 | 单条失败要单独处理 |
| 副本 | 全量导入时副本设 0，导入完恢复 | 期间无冗余 |
| refresh | 导入时 `refresh_interval: -1` | 期间不可搜 |
| translog | `durability: async`，调大 `sync_interval` | 可能丢数据 |
| 文档 ID | 用 ES 自动生成 ID | 自定义 ID 写入前要判重，多一次读 |
| Mapping | 显式映射，关 `index` / `doc_values` / `norms` | 字段能力受限 |
| 分片 | 主分片数让写入能均摊到数据节点，单分片 10–50GB | 分片过多增加集群状态开销 |
| 段 | 只读索引 force merge | 合并消耗 I/O |
| 资源 | 堆不超过物理内存一半且约 31GB 以内，剩余给文件缓存；SSD | 成本 |
| 监控 | 看 write 线程池队列与 reject、磁盘水位 | — |

## 三、分片、副本与集群

### 3.1 分片与副本【重点】

| 维度 | 主分片 Primary | 副本 Replica |
| --- | --- | --- |
| 作用 | 水平扩展容量与写入并行度 | 高可用 + 读吞吐扩展 |
| 数量 | 创建时确定，之后不可直接改 | 可随时通过 `_settings` 动态调整 |
| 分配 | 分散在不同节点 | 与主分片强制不同节点 |
| 写 | 承接写请求 | 复制主分片的写操作 |

数量选择要点：

- 7.0 起默认 **1 个主分片、1 个副本**（6.x 默认是 5 个主分片）。
- 单个分片建议 10GB 到 50GB；每 GB 堆内存对应的分片数不超过 20 个。
- 分片太少：单分片过大，恢复慢、无法利用更多节点；分片太多：文件句柄、堆内存、集群状态都变重，查询要合并更多分片结果。
- 主分片数变更只能 `reindex`、`_split`（按倍数拆）、`_shrink`（收缩），实践中就是「新建索引 + reindex + 别名切换」。
- 时序数据用 rollover + ILM 按时间或大小滚动索引，避免单索引无限增长。

### 3.2 路由机制【重点】

```
shard = hash(_routing) % number_of_primary_shards
```

- `_routing` 默认是文档 `_id`，可自定义，例如用 `userId` 作为 routing，让同一用户的数据落在同一分片。
- **因为取模，主分片数创建后不能改**，否则已有文档会散落到错误分片。
- 查询时指定 `?routing=xxx` 可以只打到一个分片，显著减少参与查询的分片数，这是自定义路由的主要收益。
- 自定义 routing 的风险是**热点分片**：某个 routing 值的数据量或请求量过大时，单分片成为瓶颈，需要设计好路由键的离散度。

### 3.3 节点类型与 node.roles【重点】

| 角色 | 职责 | 资源特点 |
| --- | --- | --- |
| `master`（候选主节点） | 管理集群状态：索引增删、Mapping、分片分配、节点上下线 | 轻量但要求稳定，不能承担重负载 |
| `data` | 存储分片，执行增删改查、搜索、聚合 | CPU、内存、磁盘 I/O 消耗大 |
| `data_content` / `data_hot` / `data_warm` / `data_cold` / `data_frozen` | 7.10 起数据角色的细分，用于冷热分层 | 按层配硬件 |
| `ingest` | 执行 ingest pipeline，写入前做字段提取、类型转换、富化 | 预处理消耗 CPU |
| `coordinating`（协调节点） | 接收请求、分发到分片、归并结果；`roles: []` 即专用协调节点 | 大查询归并消耗内存与 CPU |
| `voting_only` | 只参与投票、不当选主节点 | 用于凑多数派 |
| `remote_cluster_client` | 跨集群搜索与复制的客户端角色 | — |
| `ml` / `transform` | 机器学习、数据转换任务 | — |

- 7.9 之前用 `node.master`、`node.data` 等布尔参数，7.9 起统一为 `node.roles`；**不配置 roles 时默认拥有全部角色**。
- 生产最佳实践是**角色分离**：3 个专用候选主节点（只做 master、不存数据），满足多数派；数据节点按数据量横向扩展；查询压力大时增加专用协调节点。
- 主节点一旦承担读写负载，GC 或高负载会导致集群状态发布失败、集群不稳定，这是很常见的线上事故原因。

### 3.4 主节点选举与脑裂【了解】

- 7.x 起用自研的**基于多数派（quorum）的类 Raft 集群协调子系统**替换 Zen Discovery：选举与集群状态发布都必须得到投票节点多数派确认，第 n 个节点需要 `n/2 + 1` 票。
- **脑裂**指集群分裂成多个部分、各自选出主节点。老版本依靠 `discovery.zen.minimum_master_nodes = N/2 + 1` 规避，配错就会脑裂；7.x 后该参数被移除，多数派机制从根上避免了双主。
- 首次引导用 `cluster.initial_master_nodes` 指定候选主节点列表，只在**首次**启动时使用，后续应从配置中移除，否则可能反复触发引导。
- 候选节点必须持有最新的集群状态（term 更大）才能当选；主节点失效后，剩余候选节点重新选举，`_primary_term` 会随主分片切换递增。

### 3.5 集群健康状态【重点】

| 状态 | 含义 | 影响 |
| --- | --- | --- |
| green | 所有主分片与副本分片都已分配 | 正常 |
| yellow | 所有主分片正常，至少一个副本未分配 | 数据完整，冗余不足；单节点集群常见 |
| red | 至少一个主分片未分配 | 数据不完整，部分索引不可用 |

排查顺序：`_cluster/health` 看整体 → `_cat/shards?v` 找 `UNASSIGNED` 分片 → `_cluster/allocation/explain` 看未分配原因（节点数不足、磁盘水位、分配规则不满足）。注意 **yellow 不等于故障**，单节点集群没有第二个节点放副本，本来就该是 yellow。

### 3.6 同步复制、异步复制与高可用【重点】

| 模式 | 语义 | ES 中的体现 |
| --- | --- | --- |
| 同步复制 | 主分片与所有 in-sync 副本都写成功才返回 | 默认行为，保证在集合内的副本不落后 |
| 异步复制 | 主分片写完即返回，副本稍后同步 | 跨集群复制 CCR、以及副本写失败后的后续补偿 |

高可用要点：

- 三个专用候选主节点，主节点故障后重新选举。
- 主副分片不同节点；数据节点宕机后，主节点把该分片的 in-sync 副本提升为主分片，再在其他节点补齐副本。
- `index.unassigned.node_left.delayed_timeout` 默认 1 分钟，避免节点短暂重启就触发大规模分片迁移。
- 分片恢复基于 `_seq_no` 做**基于操作的增量同步**，不需要每次全量拷贝文件。
- 机架 / 可用区感知：`cluster.routing.allocation.awareness.attributes`，让主副分片跨故障域分布。
- 磁盘水位：低水位 85%、高水位 90% 时开始迁分片，洪水位 95% 时索引被置为只读（`index.blocks.read_only_allow_delete`），这类问题也表现为「写不进去」。
- 客户端配置多个节点地址或走负载均衡，避免单点。

### 3.7 热温冷架构、CCR 与 CCS【了解】

| 能力 | 说明 | 场景 |
| --- | --- | --- |
| 热温冷（hot / warm / cold / frozen） | 按访问频率分层：热节点放高频查询数据、温节点放低频、冷节点放归档、frozen 用共享存储挂载 | 日志、时序数据降成本 |
| ILM | 索引生命周期管理：hot → warm → cold → delete，按大小或时间 rollover | 自动化滚动与清理 |
| CCR 跨集群复制 | 主集群为 leader，备集群为 follower，由 follower 主动拉取索引变更 | 异地容灾、就近读；需商业 License |
| CCS 跨集群搜索 | 通过远程集群连接，一次查询多个集群，**不复制数据** | 多机房、多业务集群统一检索 |

## 四、检索与相关性

### 4.1 Query DSL 与 bool 查询【重点】

DSL 分叶子查询（`match`、`term`、`range`、`exists`）与复合查询（`bool`、`dis_max`、`function_score`）。

| 查询 | 是否分词 | 用途 |
| --- | --- | --- |
| `match` | 是，对查询串做分析 | 全文检索，默认 OR，可 `operator: and` |
| `match_phrase` | 是 | 短语查询，要求位置相邻 |
| `term` / `terms` | 否，精确匹配词项 | keyword、数值、ID |
| `range` | 否 | 时间、数值区间 |
| `exists` | 否 | 字段是否存在 |

`bool` 四个子句：

| 子句 | 是否打分 | 是否影响命中 | 说明 |
| --- | --- | --- | --- |
| `must` | 是 | 必须匹配 | 参与相关性打分 |
| `should` | 是 | 至少满足 `minimum_should_match` 个 | 只有 should 时默认至少 1 个 |
| `must_not` | 否 | 必须不匹配 | 在 filter 上下文执行 |
| `filter` | 否 | 必须匹配 | 不打分、可缓存，优先把过滤条件放这里 |

```json
GET /article/_search
{
  "query": {
    "bool": {
      "must":   [ { "match": { "title": "Java 面试" } } ],
      "filter": [ { "term": { "author": "meteor" } },
                  { "range": { "created": { "gte": "2026-01-01" } } } ],
      "must_not": [ { "term": { "tags": "广告" } } ],
      "should": [ { "match_phrase": { "content": "倒排索引" } } ],
      "minimum_should_match": 0
    }
  }
}
```

`text` 字段用 `term` 查整句通常查不到，因为写入时已经被分词；要查精确值请用 `keyword` 子字段。

### 4.2 filter 与查询缓存【重点】

- **查询上下文（query context）**：计算 `_score`，用于相关性排序。
- **过滤上下文（filter context）**：只判断「是否命中」，不打分，结果可被缓存，性能更好。
- 能把条件放进 `filter` 的就不要放 `must`；`term`、`range`、`exists` 放在 filter 中几乎零成本。
- 节点查询缓存（node query cache）缓存 filter 的结果位图（Roaring Bitmap），重复过滤条件直接命中。
- 分片请求缓存（shard request cache）缓存整个搜索请求的结果，要求 `size: 0` 且请求体完全相同；按时间聚合时时间范围要取整（如整天、整小时）才能命中。
- 写入会使其涉及的分片缓存失效，因此缓存对「读多写少」的索引收益最大。

### 4.3 BM25 与相关性打分【重点】

5.0 起默认相似度算法是 **BM25**，取代了 TF-IDF。核心思想：

- **TF（词频）**：词在文档中出现越多分越高，但有**饱和上限**，不会因为重复 100 次就无限加分。
- **IDF（逆文档频率）**：词越罕见越重要，停用词几乎不加分。
- **字段长度归一化**：字段越短，命中词的权重越高，由参数 `b` 控制强度。
- 公式中的 `k1` 控制词频饱和速度，`b` 控制长度归一化程度，默认 `k1=1.2`、`b=0.75`，可通过 `similarity` 自定义。

工程上的做法：

- 用 `explain: true` 或 `_explain` API 查看每个词的贡献，排查「为什么这条排前面」。
- 用 `boost` 提升字段权重，用 `function_score` 叠加业务因子（销量、点击率、发布时间衰减）。
- 用 `dis_max` / `best_fields` 让多字段中匹配最好的那个字段主导分数。
- 不需要打分的过滤条件一律放 `filter`，避免无意义的打分开销。

### 4.4 全文搜索过程与两阶段查询【重点】

1. 查询串经搜索分析器分词，得到词项。
2. 在每个相关分片本地执行：定位词项 → 取倒排列表 → 按 AND / OR 合并 → 计算得分 → 取前 `from + size` 条，返回**文档 ID + 分数 + 排序值**。
3. 协调节点归并所有分片的候选集，做全局排序，截取目标区间。
4. **fetch 阶段**：协调节点向持有目标文档的分片发起多点取回，拿到 `_source` 后返回客户端。

两点注意：

- 每个分片只返回自己的前 N 条，所以协调节点拿到的候选是 `分片数 × (from + size)`，分片越多、翻页越深，开销越大。
- IDF 是在**分片本地**统计的，多分片下打分是近似的；对打分精度敏感的场景可以只查一个分片或用 `dfs_query_then_fetch`（额外一轮，一般不用）。

### 4.5 nested 与父子关系【重点】

`object` 类型的数组会被**扁平化**：`[{a:1, b:2}, {a:3, b:4}]` 在底层变成 `a:[1,3]`、`b:[2,4]`，字段间的对应关系丢失。于是查询 `a=1 AND b=4` 会错误命中。

| 方案 | 原理 | 优点 | 代价 |
| --- | --- | --- | --- |
| `nested` | 每个嵌套对象独立成一条 Lucene 文档，查询时用 `nested` + `inner_hits` | 保留对象内字段关联，支持独立查询 | 文档数放大、更新整个父文档、查询有额外开销 |
| 父子 `join` | 一个字段 `join` 定义 parent / child，用 `has_child` / `has_parent` 查询 | 父子可独立更新，适合一对多 | 父子必须同一分片（需相同 routing），关联查询较慢 |

选型：对象数组需要**字段级关联**用 `nested`；需要**独立更新子文档**且关联层级固定用父子关系。两者都会放大索引体积，非必要不滥用。

### 4.6 场景查询（contextual search）【了解】

在关键词相关性之外叠加「当前情境」：用户地理位置、历史行为、时间、设备等。实现手段是 `function_score` 叠加距离衰减（`gauss` / `exp`）、热度、时间衰减，或用 `script_score` 自定义；也可以把用户画像作为过滤条件先缩小候选集。本质上仍是「召回 + 排序」两段式，只是排序因子从纯文本相关性扩展为业务信号。

## 五、聚合与深分页

### 5.1 聚合类型【了解】

聚合类似 SQL 的 `GROUP BY` 加聚合函数，但可多层嵌套，并与查询在同一次请求中完成。

| 类别 | 代表 | 说明 |
| --- | --- | --- |
| 指标 Metric | `avg`、`sum`、`min`、`max`、`stats`、`cardinality`、`percentiles`、`top_hits` | 对一组文档算数值指标；`cardinality` 是去重计数（HyperLogLog++，近似）；`top_hits` 取每组前几条 |
| 桶 Bucket | `terms`、`range`、`date_histogram`、`histogram`、`filters`、`nested`、`geo_distance` | 按条件分组，桶内可继续嵌套子聚合 |
| 管道 Pipeline | `derivative`、`cumulative_sum`、`avg_bucket`、`moving_fn`、`bucket_selector`、`bucket_sort` | 输入是其他聚合的输出，不是文档 |

典型场景：电商搜索页的分面导航（按品牌、价格区间统计数量）、日志按分钟统计错误数与接口耗时 P99、报表类按地区与日期汇总。

### 5.2 管道聚合【了解】

- `derivative`：对有序桶的指标求差，做环比、增长率。
- `cumulative_sum`：累计求和，做累计销量曲线。
- `moving_fn` / `moving_avg`：移动平均，平滑时间序列。
- `bucket_selector`：按条件过滤桶（如只保留均值大于阈值的桶）。
- `bucket_sort`：对桶排序并截断，常用于「取分组后前 N」。

注意管道聚合依赖桶的顺序（`date_histogram` 天然有序，`terms` 需要 `order` 明确），顺序错则结果无意义。

### 5.3 doc_values 与 fielddata【了解】

| 机制 | 存储位置 | 用途 | 说明 |
| --- | --- | --- | --- |
| `doc_values` | 磁盘列式 | 排序、聚合、脚本 | keyword、数值、日期默认开启；text 默认不支持 |
| `fielddata` | 堆内存 | 对 text 字段做排序聚合 | 极耗内存，默认关闭，不推荐开启 |
| `_source` | 原始 JSON | 取回文档、重建索引 | 可对部分字段 `enabled: false` |

- text 字段要聚合，标准做法是用 `fields` 增加一个 `keyword` 子字段，对子字段聚合。
- `eager_global_ordinals` 可在 refresh 时提前构建全局序号，加速高频 `terms` 聚合，代价是写入变慢。
- 聚合能走 doc_values 就不要用 `script`，脚本聚合无法利用列式存储且开销大。

### 5.4 聚合性能优化【了解】

- 只统计时 `size: 0`，不返回命中文档，并能命中分片请求缓存。
- 先用 `filter` 缩小范围（如时间范围），filter 不打分且可缓存。
- `terms` 聚合多分片时结果是近似的：每个分片只返回本地前 `shard_size` 个桶，可调大 `shard_size` 提高准确度，关注响应中的 `doc_count_error_upper_bound`。
- 高基数字段做全量分组用 `composite` 聚合配合 `after_key` 分页，避免一次拉回海量桶。
- 嵌套聚合用 `collect_mode: breadth_first`，先裁剪父桶再算子桶。
- `cardinality` 用 `precision_threshold` 在精度与内存之间权衡。
- `search.max_buckets`（默认 65536）与断路器防止桶过多导致 OOM。
- 时序数据用 `transform` 或 rollup 预聚合；超大离线分析交给 Spark、ClickHouse 更合适。

### 5.5 深分页的三种方案【重点】

| 方案 | 原理 | 上限 | 实时性 | 开销 | 适用 |
| --- | --- | --- | --- | --- | --- |
| `from + size` | 每个分片取前 from+size 条，协调节点归并后截断 | `index.max_result_window` 默认 10000 | 实时 | 随 from 线性增长，深翻页极慢且占堆 | 前几页、跳页 |
| `scroll` | 首次查询生成快照上下文，后续按 `scroll_id` 分批拉取 | 无硬上限 | 快照，看不到之后的写入 | 需维护上下文，占用资源，长时间不滚动会超时 | 全量导出、后台批处理 |
| `search_after` | 用上一页最后一条的排序值作为游标，只取 `size` 条 | 无硬上限 | 实时 | 每页开销恒定 | 深度翻页、无限滚动 |

`search_after` 必须配合**唯一且稳定的排序键**（如 `created desc, _id asc`），否则重复值会导致翻页重复或遗漏：

```json
GET /article/_search
{
  "size": 20,
  "sort": [ { "created": "desc" }, { "_id": "asc" } ],
  "search_after": [ 1735689600000, "doc-9527" ],
  "track_total_hits": false
}
```

补充：7.10 起推荐 `search_after` 配合 **PIT（point in time）** 使用，保证翻页期间看到一致的数据视图；`scroll` 不适合对外提供实时分页接口，因为它的上下文有状态、要显式清理。

## 六、数据同步与运维

### 6.1 MySQL 到 ES 的同步方案【重点】

| 方案 | 原理 | 优点 | 缺点 |
| --- | --- | --- | --- |
| 同步双写 | 业务写完 MySQL 再写 ES | 简单、延迟低 | 耦合业务、失败要补偿、性能受影响、易不一致 |
| MQ 异步 | 业务发消息，消费者写 ES | 解耦、可重试、削峰 | 需要可靠消息与幂等消费 |
| canal / Debezium | 订阅 binlog 解析变更 | 业务零侵入、可拿到全量与增量 | 运维成本、表结构变更需处理、删除需转换 |
| Logstash / Beats | 定时拉取或采集写入 | 配置简单 | 准实时（分钟级），不适合高频更新 |
| 全量 + 增量 | 先批量导入历史数据，再订阅增量 | 平滑上线 | 全量与增量切换点要对齐 |
| 定时对账 | 定期比对源库与 ES 的条数/校验和，补差异 | 兜底 | 只能最终一致 |

落地要点：

- 主流组合是 **canal / Debezium 订阅 binlog + Kafka 缓冲 + 消费者 bulk 写入 + 定时对账**。
- **以 MySQL 为准**，ES 只做查询副本，不一致时以源库数据修复 ES。
- 消费者要**幂等**：用业务主键作为文档 `_id`，重复消费只是覆盖；删除要显式转成 delete 操作（binlog 里是 delete 事件）。
- 全量导入时关闭 refresh、副本设 0；增量追上后再恢复。
- 关注延迟指标：binlog 位点差、Kafka lag、bulk 队列长度。

### 6.2 索引别名与滚动索引【重点】

- 别名指向一个或多个索引，对客户端暴露稳定名称；重建索引时先建新索引、导数据、原子切换别名，实现**零停机**。
- 写别名只指向一个索引（`is_write_index: true`），用于 rollover。
- **rollover** 在满足条件（时间、文档数、索引大小）时创建新索引并把写别名切过去；配合 **ILM** 自动完成 hot → warm → cold → delete。
- 索引模板（index template / composable template）统一设置分片数、Mapping、别名，新建滚动索引时自动套用。

### 6.3 快照、恢复、安全与权限【了解】

- **快照**：把索引数据与集群状态存到仓库（共享文件系统、S3、HDFS），增量备份；`_snapshot` API 创建与恢复，SLM 可做定时快照策略。
- **恢复**：可从快照恢复到同集群或其他集群，恢复时可改索引名、分片数；跨版本恢复要满足版本兼容要求。
- **安全**：X-Pack Security 提供认证与 RBAC（角色 → 权限 → 用户），支持索引级、字段级、文档级权限；节点间与客户端通信可用 TLS 加密；用 API Key 或 Token 给应用授权，避免直接使用超级用户。
- 8.x 默认开启安全功能，7.x 需显式开启；开源版安全能力有限，生产常配合网关做鉴权。

### 6.4 一致性与并发更新【重点】

- **ES 不支持跨文档事务**，只保证单个文档操作的原子性，整体是近实时、最终一致的系统。
- 并发控制用**乐观锁**：`_seq_no`（分片内递增的操作序号）+ `_primary_term`（主分片任期，每次切换递增）。更新时带上 `if_seq_no` 与 `if_primary_term`，版本不匹配返回 **409**，由客户端重读重试。
- `update` API 可设 `retry_on_conflict` 自动重试；也可用 `version_type: external` 让业务版本号（如数据库更新时间戳）决定是否允许写入，适合从 MySQL 同步。
- 主副本切换后，靠 `_seq_no` 与全局 / 本地检查点识别旧主遗留的过时操作，只同步差异部分。
- 写入默认约 1 秒后可搜；需要更强可见性用 `refresh=wait_for`，但会拉高写入延迟。
- 真正需要事务的业务以数据库为准，ES 通过 binlog / MQ 做最终一致，并用对账补偿缺口。

## 七、高频面试题与自测清单

### 7.1 高频面试题

| # | 题目 | 答案要点 | 章节 |
| --- | --- | --- | --- |
| 1 | 什么是节点和集群？ | 节点是一个 ES 实例，存数据并参与索引搜索；集群是多节点共享数据的整体 | 一 |
| 2 | 文档和 type 是什么关系，type 还在吗？ | 文档是最小可搜索单元；type 是早期逻辑分类，7.0 已移除，写入统一用 `_doc` | 一 |
| 3 | 倒排索引是什么，由哪几部分组成？ | 词项到文档列表的映射；由词项字典、FST 词项索引、倒排列表（ID、TF、位置、偏移）组成 | 一 |
| 4 | FST 和 Frame of Reference 分别解决什么问题？ | FST 压缩词项字典常驻内存加速定位；FOR 对有序文档 ID 做差值按位压缩省磁盘 | 一 |
| 5 | 分析器由哪些组件构成？ | 字符过滤器 0 到多个 + 分词器 1 个 + 词元过滤器 0 到多个，按序执行 | 一 |
| 6 | text 和 keyword 的区别？ | text 分词可全文检索但不能直接排序聚合；keyword 不分词用于精确匹配、排序聚合 | 一 |
| 7 | Mapping 怎么设计，能改类型吗？ | 显式映射；不搜索 `index:false`、不聚合关 `doc_values`；类型不可改，需 reindex + 别名切换 | 一 |
| 8 | 为什么说是近实时？ | 写入先入 buffer 与 translog，默认 1 秒 refresh 生成新段后才可搜 | 一 |
| 9 | 写一条文档的完整流程？ | 协调节点路由 → 主分片写入 buffer/translog → 并行复制到 in-sync 副本 → 全部确认后返回 | 二 |
| 10 | refresh、flush、translog 的区别？ | refresh 管可见不落盘；flush 提交段并清空 translog；translog 保证崩溃不丢数据 | 二 |
| 11 | segment 不可变带来什么利弊？ | 无锁读、可缓存；更新删除靠标记删除与后台合并，合并消耗 I/O | 二 |
| 12 | 写入性能怎么优化？ | bulk 5–15MB、调大或关闭 refresh、导入期副本 0、translog async、自动 ID、精简 Mapping、force merge、SSD | 二 |
| 13 | 分片和副本各起什么作用？ | 分片水平扩展容量与并行度；副本提供高可用与读扩展，且不与主分片同节点 | 三 |
| 14 | 主分片数为什么不能改？ | 路由公式 `hash(routing) % 主分片数` 依赖它；只能 reindex、split、shrink | 三 |
| 15 | 路由机制是怎样的？ | 默认用文档 ID 哈希取模；可自定义 routing 让同组数据同分片，查询指定 routing 减少分片 | 三 |
| 16 | 有哪些节点类型？ | master 管集群状态、data 存数据、coordinating 归并、ingest 预处理；7.9 起用 `node.roles` | 三 |
| 17 | 主节点怎么选举，脑裂怎么避免？ | 7.x 基于多数派 quorum 的类 Raft 协调；不再需要 `minimum_master_nodes` | 三 |
| 18 | green、yellow、red 分别代表什么？ | 全部分片正常 / 主分片正常但副本缺失 / 主分片缺失数据不完整 | 三 |
| 19 | 同步复制和异步复制区别？ | 同步要所有 in-sync 副本确认才成功；异步主分片写完即返回，副本稍后同步 | 三 |
| 20 | ES 怎么保证高可用？ | 3 个候选主节点、副本跨节点提升、延迟分配 1 分钟、机架感知、快照与 CCR | 三 |
| 21 | match 和 term 的区别？ | match 会分析查询串做全文匹配；term 不分词精确匹配词项，适合 keyword 与数值 | 四 |
| 22 | bool 查询四个子句的作用？ | must 打分必中、should 可选打分、must_not 必不中、filter 必中不打分可缓存 | 四 |
| 23 | 为什么 filter 更快？ | 不计算相关性分数，结果位图可进节点查询缓存，请求可进分片请求缓存 | 四 |
| 24 | BM25 相比 TF-IDF 好在哪？ | 词频有饱和上限、结合 IDF 与字段长度归一化，避免长文档和堆词刷分 | 四 |
| 25 | 一次搜索是怎么执行的？ | query 阶段各分片本地检索返回 ID 与分数，协调节点归并，fetch 阶段取回 `_source` | 四 |
| 26 | object 数组为什么会查错，怎么解决？ | 数组被扁平化丢失字段关联；用 nested 类型 + nested 查询，或用父子 join | 四 |
| 27 | 聚合有哪些类型？ | 指标（avg、cardinality、top_hits）、桶（terms、date_histogram）、管道（derivative、bucket_selector） | 五 |
| 28 | 为什么 text 字段聚合要用 keyword 子字段？ | text 无 doc_values 且分词后语义不对，需 keyword 子字段提供列式存储 | 五 |
| 29 | 深分页三种方案怎么选？ | from+size 上限 10000 且越深越慢；scroll 适合导出；search_after 配 PIT 适合实时深翻页 | 五 |
| 30 | MySQL 数据怎么同步到 ES？ | 同步双写、MQ 异步、canal/Debezium 订阅 binlog、Logstash；主流是 binlog + Kafka + 幂等 bulk + 对账 | 六 |
| 31 | 索引别名和滚动索引有什么用？ | 别名支持零停机重建与原子切换；rollover 按大小时间滚动新索引，配合 ILM 分层与清理 | 六 |
| 32 | ES 支持分布式事务吗，并发怎么控制？ | 不支持跨文档事务，单文档原子、整体最终一致；用 `if_seq_no` + `if_primary_term` 乐观锁，冲突返回 409 | 六 |
| 33 | 快照和安全怎么做？ | 仓库增量快照 + SLM 定时；X-Pack RBAC、TLS、API Key，支持索引/字段/文档级权限 | 六 |
| 34 | CCR 和 CCS 有什么区别？ | CCR 把索引异步复制到另一集群做主备容灾；CCS 不复制数据，只做跨集群联合查询 | 三 |

### 7.2 动手实验

- [ ] 建一个 3 主 1 副的索引，用 `_cat/shards?v` 观察主副分片的节点分布
- [ ] 用 `_analyze` 对比 `standard` 与 `ik_smart` 对同一句中文的分词结果
- [ ] 写入一条文档后立刻查询、再 `refresh=wait_for` 查询，验证近实时与强制刷新
- [ ] 对同一索引分别用 `must` 与 `filter` 写同样的 `term` 条件，用 `profile` 对比耗时与缓存
- [ ] 构造 object 数组并查询错配组合，再用 nested 类型复现并修复
- [ ] 用 `from=9990&size=20` 触发结果窗口异常，再用 `search_after` + PIT 翻到第 1000 条
- [ ] 用 `terms` 聚合对比不同 `shard_size` 下的 `doc_count_error_upper_bound`
- [ ] 停掉一个数据节点，观察健康状态从 green 变 yellow、分片重新分配后恢复的过程

### 7.3 自测清单

- [ ] 能画出倒排索引的三层结构，并说明 FST、FOR、跳表各自解决什么问题
- [ ] 能解释分析器三组件与「索引与搜索分析器不一致导致搜不到」
- [ ] 能完整讲出写入链路，并区分 refresh、flush、translog 的职责
- [ ] 能说出至少 6 条写入性能优化手段及其代价
- [ ] 能默写路由公式，并解释主分片数为何不可改
- [ ] 能说明 master、data、coordinating、ingest 的职责与生产部署建议
- [ ] 能解释 7.x 多数派选举如何避免脑裂
- [ ] 能区分 match 与 term，并说清 bool 四个子句与 filter 缓存
- [ ] 能说清 BM25 的三个因子与 `explain` 排查方法
- [ ] 能对比深分页三种方案的原理、上限与适用场景
- [ ] 能设计一套 MySQL → ES 的同步方案，并说明幂等、删除、对账、延迟监控
- [ ] 能结合项目讲一次搜索或日志分析的索引设计与调优
