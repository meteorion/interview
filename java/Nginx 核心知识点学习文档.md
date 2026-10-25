# Nginx 核心知识点学习文档

Oct 25, 2026 · @meteorzhong

## 导读

本文展开《知识图谱.html》题库中 Nginx 分类的 30 道题。图谱目前没有单独的 Nginx 节点，级别按题目在面试中的实际重要度标注：进程模型与高并发原理、请求处理阶段、location 匹配、反向代理与负载均衡、静态资源优化、HTTPS 加固、限流与高可用为**【重点】**；内存池、模块系统、日志细节、重写规则等运维类内容为**【了解】**。默认版本为 Nginx 1.24+（`nginx.conf` 语法在各版本间基本稳定）。

| 知识模块 | 级别 | 本文章节 |
| --- | --- | --- |
| 进程模型、事件驱动、内存管理 | 重点 | 一、进程模型与高并发原理 |
| 配置层级、请求处理阶段、location 匹配 | 重点 | 二、配置结构与请求处理 |
| 反向代理、upstream、负载均衡、WebSocket | 重点 | 三、反向代理与负载均衡 |
| sendfile、gzip、浏览器缓存、proxy_cache | 重点 | 四、静态资源与缓存 |
| HTTPS、TLS 加固、访问控制、安全配置 | 重点 | 五、HTTPS 与安全 |
| 限流、超时、高可用、日志与排障 | 重点 | 六、限流、高可用与运维 |
| 模块系统、内存池、重写规则、内存泄漏 | 了解 | 一、二、六中的对应小节 |

**复习顺序**：进程模型 → 请求处理阶段 → location 匹配 → 反向代理与负载均衡 → 静态资源与缓存 → HTTPS 与安全 → 限流与高可用 → 排障。面试主线是「一个请求在 Nginx 里怎么走完」，把第二章的 11 个阶段背熟，其余章节都是往这条主线上挂细节。

## 一、进程模型与高并发原理

### 1.1 master 与 worker【重点】

Nginx 启动后是一个 master 加多个 worker 的多进程模型：

| 进程 | 职责 |
| --- | --- |
| master | 读取和校验配置、创建监听套接字、fork 并管理 worker、接收信号做平滑重载与热升级、worker 异常退出时重新拉起 |
| worker | 真正处理连接与请求，彼此独立，一个 worker 崩溃不影响其他 worker |
| cache manager / cache loader | 管理 proxy_cache 的缓存文件、启动时加载缓存索引，由 master 派生 |

- `worker_processes auto`：等于 CPU 核数；计算密集型（大量 gzip、HTTPS 握手）可等于或略多于核数，纯反向代理可略少。
- `worker_cpu_affinity`：把 worker 绑到固定核，减少进程迁移与缓存失效，但对容器环境（CPU 配额而非独占核）收益有限，需实测。
- `worker_rlimit_nofile`：抬高单个 worker 可打开的文件描述符上限，需与系统 `ulimit -n` 一致。
- `user`、`pid`、`daemon`、`error_log` 属于全局（main）配置。

多进程而不是多线程的原因：各 worker 地址空间独立，避免锁竞争和线程安全问题，同时又能利用多核；代价是共享数据必须走共享内存。

### 1.2 事件驱动与 IO 多路复用【重点】

每个 worker 是**单线程的事件循环**：注册连接上关心的事件，`epoll_wait` 阻塞等待，有事件就绪就回调对应 handler，处理完继续等下一个事件。任何可能等待的操作（等客户端发数据、等上游响应）都不会阻塞 worker，而是注册事件后先去处理别的连接。

**为什么 epoll 高效**：

| 机制 | 连接上限 | 找就绪连接 | 数据拷贝 |
| --- | --- | --- | --- |
| select | 1024（`fd_set` 位图） | 遍历全部 O(n) | 每次把整个集合拷进内核 |
| poll | 无（链表） | 遍历全部 O(n) | 同上 |
| epoll | 无 | 内核回调把就绪 fd 放入就绪队列，只返回就绪的 | fd 只在注册时拷一次（红黑树） |

所以 epoll 的开销与**活跃连接数**相关，而与总连接数无关；再加上每个连接只占很少内存（连接结构体 + 读写缓冲，按需分配），单机支撑数万到数十万连接才成立。Linux 用 epoll，BSD/macOS 用 kqueue，Windows 用 IOCP。

**边缘触发（ET）与水平触发（LT）**：epoll 默认 LT，只要缓冲区还有数据就一直通知；ET 只在状态变化时通知一次，必须一次读到 `EAGAIN` 为止，否则数据会滞留。Nginx 的 epoll 传输模式使用 ET 以获得更高性能。

**网络 I/O 与磁盘 I/O 的区别【重点】**：网络 I/O 天然适合非阻塞 + 多路复用；但磁盘 I/O（`read` 静态文件）在 Linux 上无法真正非阻塞，会阻塞 worker。因此 Nginx 提供 `aio threads`：把读文件交给线程池，worker 事件循环不被拖住。`directio` 让大文件绕过页缓存直接读，避免污染缓存。

### 1.3 惊群与连接分配【重点】

多个 worker 同时监听同一端口，新连接到来时内核会唤醒所有阻塞在 `accept` 上的 worker，但只有一个能成功，其余被唤醒后空跑——这就是**惊群（thundering herd）**。

| 方案 | 做法 | 说明 |
| --- | --- | --- |
| `accept_mutex on` | worker 之间用共享锁串行 `accept` | 传统方案，锁本身也有开销，默认关闭 |
| `reuseport` | `listen 80 reuseport` | 内核按四元组哈希把连接直接分给某个 worker 的独立监听队列，**新版本推荐**，无锁无惊群 |

### 1.4 并发上限与系统参数【重点】

- 理论上限：`worker_processes × worker_connections`（`worker_connections` 默认 512，生产常调到 10240 以上）。
- **反向代理要减半**：一个客户端请求在 Nginx 侧占用两条连接（下游客户端 + 上游服务端）。
- 必须同步调整的系统参数：

| 参数 | 作用 |
| --- | --- |
| `ulimit -n` / `worker_rlimit_nofile` | 文件描述符上限，否则连接数到了也打不开新连接 |
| `net.core.somaxconn` 与 `listen ... backlog` | 全连接队列长度，太小会丢握手 |
| `net.ipv4.tcp_max_syn_backlog` | 半连接队列长度 |
| `net.ipv4.ip_local_port_range` | 作为客户端连上游时可用端口范围，反向代理高并发时会耗尽 |
| `net.ipv4.tcp_tw_reuse` | 允许复用 TIME_WAIT 连接（客户端侧） |

### 1.5 内存池与共享内存【了解】

- **内存池**：每个连接、每个请求都有自己的内存池，申请小块内存时从池里按页分配，请求结束整池释放，避免频繁 `malloc/free` 与碎片；大块内存直接走系统分配。这是「按生命周期批量释放」的思路，和 Netty 的池化 ByteBuf 目标相同、手段不同。
- **共享内存**：`shared` 区域用于跨 worker 共享数据，如 SSL 会话缓存（`ssl_session_cache shared:SSL:10m`）、`limit_req` / `limit_conn` 的状态、upstream 的 `zone`（多 worker 共享负载与健康状态）。共享内存用自旋锁 + 红黑树/队列管理，分配后由 slab 分配器管理。
- **资源上限**：`worker_connections`、缓冲区大小（`client_body_buffer_size`、`proxy_buffer_size`）、`*_buffers` 数量都直接影响内存占用，调优时要按「连接数 × 单连接缓冲」估算。

### 1.6 模块系统【了解】

Nginx 功能全部由模块拼装：核心模块、事件模块、HTTP 模块、邮件/流模块、第三方模块。每个模块提供自己的配置指令（`ngx_command_t`），并在请求处理阶段挂载 handler。

- **静态模块**：编译时用 `--with-xxx` 决定是否包含，体积小但改功能要重新编译。
- **动态模块**：`--add-dynamic-module` 编译出 `.so`，在配置里用 `load_module modules/xxx.so` 加载，官方从 1.9.11 起支持。注意动态模块与 Nginx 主程序版本/编译参数必须一致。
- 典型第三方模块：`ngx_http_brotli_filter_module`（brotli 压缩）、`nginx-echo`（调试）、`lua-nginx-module`（OpenResty 基础）。

## 二、配置结构与请求处理

### 2.1 配置层级与继承【重点】

```
main（全局）
├── events { worker_connections 10240; }
└── http {
    ├── upstream backend { ... }
    ├── server {                      # 虚拟主机
    │   ├── server_name example.com;
    │   └── location /api/ { ... }    # 路由与处理
    │   }
    └── include /etc/nginx/conf.d/*.conf;
}
```

- **继承规则**：内层没有显式声明时继承外层值；一旦内层声明，就整体覆盖（不是逐项合并）。`add_header` 的继承尤其容易踩坑——在 `location` 里写 `add_header` 会丢掉 `server` 级的 `add_header`（除非用 `add_header_inherit merge`（1.29+）或 `always` 参数配合）。
- **`include`**：把配置拆文件，按通配符顺序加载。
- **`root` 与 `alias` 的区别【重点】**：`root` 把 URI 拼到路径后面（`root /data;` + `/img/a.png` → `/data/img/a.png`）；`alias` 用 alias 路径**替换** location 匹配到的部分（`location /img/ { alias /data/static/; }` + `/img/a.png` → `/data/static/a.png`）。alias 结尾的斜杠要与 location 保持一致，否则会出现路径拼接错误。

### 2.2 请求处理的 11 个阶段【重点】

一个 HTTP 请求在 Nginx 内部按固定顺序经过 11 个 phase，模块把 handler 挂在某个 phase 上：

| 阶段 | 作用 | 典型模块/指令 |
| --- | --- | --- |
| POST_READ | 读完请求头后立即执行 | `realip`（从 X-Forwarded-For 取真实 IP） |
| SERVER_REWRITE | server 级 rewrite | `rewrite`、`return` |
| FIND_CONFIG | 按 URI 匹配 location | 匹配规则见 2.3 |
| REWRITE | location 级 rewrite | `rewrite`、`set` |
| POST_REWRITE | URI 被改写后重新匹配 location | 内部循环，最多 10 次 |
| PREACCESS | 访问前限流 | `limit_req`、`limit_conn`、`max_ranges` |
| ACCESS | 访问控制 | `allow` / `deny`、`auth_basic`、`auth_request` |
| POST_ACCESS | 访问控制后的收尾 | `satisfy` |
| PRECONTENT | 内容前的准备工作 | `try_files`、`mirror`、`index` |
| CONTENT | 生成响应内容 | `static`、`proxy_pass`、`fastcgi_pass`、`return` |
| LOG | 记录日志 | `access_log` |

**要点**：`return` 和 `rewrite` 在 REWRITE 阶段就能结束流程，`limit_req` 在 PREACCESS 阶段拒绝请求（早于 CONTENT，避免打到上游），`try_files` 在 PRECONTENT 阶段判断文件是否存在再决定走静态还是转发。

### 2.3 location 匹配优先级【重点】

| 修饰符 | 含义 | 优先级 |
| --- | --- | --- |
| `=` | 精确匹配，URI 完全相同 | 最高，命中即停止 |
| `^~` | 前缀匹配，命中后**不再尝试正则** | 次高 |
| `~` | 正则匹配，区分大小写 | 按配置顺序，第一个命中即用 |
| `~*` | 正则匹配，不区分大小写 | 同上 |
| 无修饰符 | 普通前缀匹配，取**最长**匹配 | 最低 |

执行顺序：先找 `=` 精确匹配 → 再找最长前缀匹配，若该前缀带 `^~` 则直接用 → 否则按配置顺序试正则，命中即用 → 正则都没命中则用最长前缀匹配。

```nginx
location = /50x.html { root /usr/share/nginx/html; }   # 精确
location ^~ /static/ { root /data; }                    # 前缀且不试正则
location ~* \.(jpg|png|css|js)$ { expires 30d; }        # 正则，忽略大小写
location / { proxy_pass http://backend; }               # 兜底
```

**常见坑**：正则 location 写在前面前缀后面前会影响结果；`location /api` 会匹配 `/api`、`/apixxx`，而 `location /api/` 只匹配 `/api/` 开头，需要精确边界时用 `^~ /api/` 或正则 `^/api(/|$)`。

### 2.4 server_name 与多站点【重点】

同一 `listen` 地址端口下可以有多个 `server` 块，靠 `server_name` 选择虚拟主机。匹配顺序：精确域名 → 以 `*` 开头的通配 → 以 `*` 结尾的通配 → 正则（`~` 开头，按顺序）→ `default_server`。

- 未匹配到任何 `server_name` 时，使用该 `listen` 上的 `default_server`（未显式指定则用第一个 server 块）。
- 支持多域名：`server_name example.com www.example.com;`
- 支持 SNI：同一个 IP 的 443 端口可按域名下发不同证书，配置在不同 `server` 的 `ssl_certificate` 上。

```nginx
server {
    listen 80 default_server;
    server_name _;
    return 444;              # 未匹配域名的非法请求直接断开
}
```

### 2.5 rewrite 与 return【了解】

- `rewrite 正则 替换 [flag]`，flag 有 `last`（重新匹配 location）、`break`（在本 location 内继续）、`redirect`（302）、`permanent`（301）。
- `return code [text|url]`：更轻量、语义清晰，**能用 return 就不要用 rewrite**。
- 大量重写规则的优化：规则按命中频率排序、正则尽量锚定（`^`）并避免灾难性回溯、相同前缀的规则合并、把重定向逻辑上移到网关；`rewrite_log on` 只用于调试。
- 循环保护：`rewrite` 触发的内部跳转最多 10 次，超过返回 500。

### 2.6 try_files 与动静分离【重点】

```nginx
location / {
    root /data/www;
    try_files $uri $uri/ /index.html;   # SPA 前端路由回退
}

location ~ \.php$ {
    try_files $uri =404;                # 文件不存在直接 404，避免把不存在的脚本交给上游
    fastcgi_pass 127.0.0.1:9000;
}
```

动静分离：静态资源走独立 location（甚至独立域名 + CDN），动态请求 `proxy_pass` 到上游。这样静态请求不占用上游连接，日志与缓存策略也能分开配置。路径穿越防护：`root` 与 `alias` 配置正确即可，Nginx 会规范化 URI；自己实现下载服务时要显式校验 `..`。

## 三、反向代理与负载均衡

### 3.1 proxy_pass 的关键差异【重点】

`proxy_pass` 后面**有没有 URI** 决定是否改写路径：

| 写法 | 请求 `/api/user` 转发到 |
| --- | --- |
| `proxy_pass http://backend;`（无 URI） | `http://backend/api/user`，原样透传 |
| `proxy_pass http://backend/;`（有 `/`） | `http://backend/user`，location 匹配部分被替换为 `/` |
| `proxy_pass http://backend/v1/;` | `http://backend/v1/user` |

规则：proxy_pass 带 URI 时，用该 URI 替换 location 匹配到的前缀；不带 URI 时原样转发。用变量拼 URL（`proxy_pass http://$upstream;`）时，必须自己用 `rewrite` 处理路径。

### 3.2 upstream 与负载均衡算法【重点】

```nginx
upstream backend {
    least_conn;
    server 10.0.0.11:8080 weight=3 max_fails=2 fail_timeout=10s;
    server 10.0.0.12:8080 backup;
    keepalive 64;
}
```

| 算法 | 指令 | 适用 |
| --- | --- | --- |
| 轮询（默认） | 无 | 后端性能一致 |
| 加权轮询 | `weight=N` | 机器配置不同 |
| 最少连接 | `least_conn` | 请求耗时差异大 |
| IP 哈希 | `ip_hash` | 需要按客户端会话粘滞（IPv4 按前三段，IPv6 按前三段） |
| 一致性哈希 | `hash $request_uri consistent` | 缓存命中率要求高（如分片缓存） |
| 随机 | `random two` | 大规模集群、降低重试时打同一台的概率 |

- `backup`：备用机，主机组全挂才启用；`down`：手动摘除。
- 会话保持也可以用 Cookie 方案（`sticky` 需第三方模块），或用上游自己的分布式 Session。
- 多 worker 共享 upstream 状态需要 `zone backend_zone 64k;`，否则各 worker 的健康统计互不可见。

### 3.3 健康检查与失败重试【重点】

- **被动健康检查**：`max_fails=2 fail_timeout=10s` —— 10 秒内失败 2 次就标记不可用，10 秒后再试。Nginx 开源版只有被动检查。
- **主动健康检查**：`health_check interval=5s fails=3 passes=2 uri=/health;`（Nginx Plus 商业版）。
- **重试**：`proxy_next_upstream error timeout http_502 http_503 http_504;` 决定什么情况下换下一台重试；`proxy_next_upstream_tries`、`proxy_next_upstream_timeout` 限制次数与总时长。
- **重试的坑**：非幂等请求（POST 下单）重试可能造成重复提交，需要上游幂等；请求体已被读取后无法重试（`proxy_request_buffering` 相关）。

### 3.4 头部透传与连接复用【重点】

```nginx
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_http_version 1.1;
proxy_set_header Connection "";
```

- 不透传 `Host` 会让上游拿到的是 upstream 名字而不是真实域名，多租户/虚拟主机场景直接 404。
- `X-Forwarded-For` 是可伪造的，最外层网关要把真实 IP 写入可信头（如 `X-Real-IP`），`real_ip` 模块配合 `set_real_ip_from` 只信任已知代理。
- **`proxy_http_version 1.1` + `Connection ""` 才能启用 upstream keepalive**；默认 HTTP/1.0 且 `Connection: close`，每条请求都重新握手，高并发下 TIME_WAIT 暴涨。`keepalive 64` 指每个 worker 与上游保持的空闲连接数。
- TraceId / Token 透传：用 `proxy_set_header X-Trace-Id $request_id;` 或透传上游传入的头，保证链路追踪不断裂。

### 3.5 超时与缓冲【重点】

| 指令 | 含义 | 常见取值 |
| --- | --- | --- |
| `proxy_connect_timeout` | 与上游建立连接的超时 | 3–5s |
| `proxy_send_timeout` | 向上游发送请求的超时（两次写之间） | 10–60s |
| `proxy_read_timeout` | 等待上游响应的超时（两次读之间） | 30–60s，长连接/流式接口放大 |
| `client_body_timeout` / `client_header_timeout` | 读客户端请求体/头的超时 | 10–30s |
| `send_timeout` | 向客户端发送响应的超时 | 10–60s |
| `keepalive_timeout` | 客户端连接空闲保持时间 | 15–75s |

- **超时要逐层递减**：客户端 > Nginx > 上游网关 > 具体服务，否则前端已断开、后端还在算。
- **缓冲**：`proxy_buffering on`（默认）让 Nginx 先把上游响应读到缓冲区再慢慢发给慢客户端，保护上游；`proxy_buffers` / `proxy_buffer_size` 控制大小，`proxy_max_temp_file_size` 限制落盘。流式接口（SSE、大文件下载）必须 `proxy_buffering off`，否则数据被攒住不吐。

### 3.6 WebSocket 与其他协议代理【重点】

```nginx
location /ws/ {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 3600s;
}
```

- WebSocket 是 HTTP/1.1 的协议升级（`101 Switching Protocols`），必须显式透传 `Upgrade`/`Connection` 并保持 1.1。
- 空闲长连接会被 `proxy_read_timeout` 掐断，需要放大超时或让客户端定时发心跳（ping/pong）。
- HTTP/2 到上游（`grpc_pass`）需要 `http2` 支持；`stream` 模块用于四层 TCP/UDP 代理（数据库、MQ、DNS）。

## 四、静态资源与缓存

### 4.1 sendfile 与内核零拷贝【重点】

| 指令 | 作用 |
| --- | --- |
| `sendfile on` | 文件内容在内核中直接从页缓存送到套接字，省去「内核→用户→内核」两次拷贝与两次上下文切换 |
| `tcp_nopush on` | 与 sendfile 配合，把响应头和文件开头凑满一个 MSS 再发，减少小包 |
| `tcp_nodelay on` | 长连接上关闭 Nagle，降低小数据延迟（与 nopush 场景相反，Nginx 按场景自动切换） |
| `sendfile_max_chunk 1m` | 限制单次 sendfile 传输量，避免一个大文件占满 worker |
| `aio threads` | 读文件交给线程池，避免磁盘 I/O 阻塞事件循环 |
| `directio 4m` | 大于该大小的文件绕过页缓存，避免大文件把缓存冲掉 |

**open_file_cache**：缓存文件描述符、大小、修改时间，减少每次请求的 `open` + `stat` 系统调用。

```nginx
open_file_cache max=10000 inactive=60s;
open_file_cache_valid 30s;
open_file_cache_min_uses 2;
open_file_cache_errors on;
```

### 4.2 gzip 与预压缩【重点】

```nginx
gzip on;
gzip_comp_level 5;
gzip_min_length 1k;
gzip_types text/plain text/css application/json application/javascript image/svg+xml;
gzip_vary on;
gzip_static on;      # 直接发送构建时预生成的 .gz，节省实时压缩的 CPU
```

- `gzip_comp_level` 1–9，6 以上压缩率提升有限但 CPU 增长明显，生产常用 4–6。
- 已压缩格式（jpg/png/mp4/zip）不要再 gzip。
- `gzip_static` 需要构建阶段生成 `.gz` 文件（前端构建插件或 `zopfli`），高 QPS 站点收益最大。
- brotli（`ngx_brotli`）比 gzip 压缩率高约 15–20%，但需额外模块与预压缩文件，且只在 HTTPS 下被主流浏览器接受（`brotli_static` + `brotli_types`）。

### 4.3 浏览器缓存与协商缓存【重点】

| 资源类型 | 策略 |
| --- | --- |
| 带哈希指纹的 JS/CSS/图片 | `expires 1y;` + `Cache-Control: public, max-age=31536000, immutable` |
| HTML 入口 | `Cache-Control: no-cache`（可缓存但每次协商），保证发版立即生效 |
| 接口响应 | 一般 `no-store`，或按业务设置短缓存 |

```nginx
location ~* \.(js|css|png|jpg|woff2)$ {
    expires 1y;
    add_header Cache-Control "public, max-age=31536000, immutable";
}
```

Nginx 默认发送 `ETag` 与 `Last-Modified`，浏览器带 `If-None-Match` / `If-Modified-Since` 时返回 `304`，省掉响应体。`etag off` 可关闭（多台机器 `mtime` 不一致时不建议依赖 ETag）。

### 4.4 proxy_cache【重点】

```nginx
proxy_cache_path /data/cache levels=1:2 keys_zone=api_cache:100m inactive=10m max_size=10g;
server {
    location /api/ {
        proxy_cache api_cache;
        proxy_cache_key "$scheme$request_method$host$request_uri";
        proxy_cache_valid 200 302 60s;
        proxy_cache_valid 404 10s;
        proxy_cache_use_stale error timeout updating http_500 http_502 http_503;
        proxy_cache_lock on;            # 回源串行化，防缓存击穿
        add_header X-Cache-Status $upstream_cache_status;
    }
}
```

- `keys_zone` 只存键和元数据（1MB 约 8000 个键），实际内容存磁盘；`max_size` 触发 LRU 淘汰。
- **`proxy_cache_lock`** 是关键：并发未命中时只放一个请求回源，其余等待，避免缓存击穿。
- **`proxy_cache_use_stale`** 让上游故障时继续返回过期缓存，是降级手段。
- 缓存清理：开源版只能删文件 + `reload`，或按 `proxy_cache_purge`（第三方模块）；更常用的是缩短过期时间或用带版本号的 key。

## 五、HTTPS 与安全

### 5.1 TLS 握手与证书链【重点】

```nginx
server {
    listen 443 ssl;
    http2 on;
    server_name example.com;

    ssl_certificate     /etc/nginx/cert/fullchain.pem;   # 服务器证书 + 中间证书
    ssl_certificate_key /etc/nginx/cert/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-CHACHA20-POLY1305;
    ssl_prefer_server_ciphers on;

    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    ssl_stapling on;
    ssl_stapling_verify on;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
}

server {
    listen 80;
    return 301 https://$host$request_uri;
}
```

握手过程：协商版本与套件 → 服务端下发证书 → 客户端校验证书链与域名 → ECDHE 交换出会话密钥 → 后续用对称加密传输。TLS 1.2 完整握手需 2-RTT，TLS 1.3 只需 1-RTT，并且只保留支持前向保密的套件。

**证书链**：`ssl_certificate` 必须是「服务器证书 + 中间证书」的完整链，缺中间证书会让部分客户端（尤其 Android）校验失败。

### 5.2 安全加固要点【重点】

| 风险 | 措施 |
| --- | --- |
| 降级与 SSL 剥离 | HSTS（`max-age` + `includeSubDomains` + `preload`） |
| 弱协议/套件 | 只启用 TLS 1.2/1.3，禁用 RC4/3DES/MD5，TLS 1.2 下 `ssl_prefer_server_ciphers on` |
| 前向保密缺失 | 关闭 `ssl_session_tickets` 或定期轮换 ticket 密钥；禁用 RSA 密钥交换 |
| 证书吊销状态查询慢 | OCSP Stapling，由服务端代取并缓存 |
| 版本信息泄露 | `server_tokens off`，并自定义错误页 |
| 危险 HTTP 方法 | `limit_except` 只放行 GET/POST |
| 目录与文件泄露 | 禁止访问 `.git`、`.env`、备份文件；`autoindex off` |
| 点击劫持/XSS 辅助 | `X-Frame-Options`、`X-Content-Type-Options`、CSP |
| 私钥保护 | 私钥权限 600、与证书分离存放、监控到期时间 |

```nginx
location ~ /\.(git|env|svn) { deny all; }
limit_except GET POST { deny all; }
add_header X-Content-Type-Options nosniff always;
add_header X-Frame-Options SAMEORIGIN always;
```

注意 `add_header` 只在 200/204/301/302/304 等状态码生效，错误响应要加 `always` 参数。

### 5.3 TLS 性能优化【重点】

1. **会话复用**：`ssl_session_cache shared:SSL:10m`（约 4 万会话）+ 合理的 `ssl_session_timeout`，把完整握手降为 1-RTT/0-RTT；多 worker 必须用 `shared` 而不是 `builtin`。
2. **套件选择**：ECDHE + AES-GCM/ChaCha20；没有 AES-NI 的机器用 ChaCha20 更快。
3. **OCSP Stapling**：省掉客户端单独查询吊销状态的一次连接。
4. **缓冲区**：`ssl_buffer_size 4k` 降低小响应的延迟（默认 16k，适合大文件）。
5. **硬件与卸载**：AES-NI、QAT 卡，或在 L4 负载均衡/云 LB 上终止 TLS。
6. **HTTP/2**：多路复用减少连接数；注意 HTTP/2 下 `keepalive_requests` 的作用变化。
7. **证书算法**：ECDSA 证书比 RSA 握手更快、体积更小。

### 5.4 访问控制与认证【重点】

```nginx
location /admin/ {
    allow 10.0.0.0/8;
    allow 192.168.1.0/24;
    deny  all;                    # 顺序敏感：先匹配先生效
    auth_basic "restricted";
    auth_basic_user_file /etc/nginx/.htpasswd;
}
```

- `allow` / `deny` **按书写顺序匹配，命中即生效**，所以 `deny all` 放在最后；也可放 `http` 或 `server` 级。
- `auth_request` 子请求适合把鉴权交给统一认证服务（返回 2xx 放行、401/403 拒绝），比在每个服务里重复实现更集中。
- IP 控制只适合内网或临时限制，公网场景应结合 WAF、网关鉴权，且注意 `X-Forwarded-For` 伪造问题。

## 六、限流、高可用与运维

### 6.1 limit_req 漏桶限流【重点】

```nginx
limit_req_zone $binary_remote_addr zone=req_per_ip:10m rate=10r/s;
limit_req_zone $server_name zone=req_global:10m rate=1000r/s;

location /api/ {
    limit_req zone=req_per_ip burst=20 nodelay;
    limit_req_status 429;
}
```

| 参数 | 含义 |
| --- | --- |
| `rate=10r/s` | 漏桶出水速率，即平均允许速率 |
| `burst=20` | 允许的突发队列长度（桶容量），超出直接拒绝 |
| `nodelay` | 突发部分立即处理，不排队延迟；不加则按速率排队，平滑但增加 RT |
| `zone` 大小 | 10m 约存 16 万个 IP 状态；键用 `$binary_remote_addr` 比 `$remote_addr` 省内存 |

- 默认超限返回 503，建议改为 429（语义正确）并配合前端重试提示。
- 按用户限流可用 `$http_x_user_id` 或 `$jwt_claim`（需模块），按接口可用 `$uri`。
- 局限：单机限流，集群需要网关层（Redis + Lua）做全局限流；`limit_req` 是**漏桶**，不允许多余突发通过，和令牌桶的语义不同。

### 6.2 limit_conn 与连接数控制【重点】

```nginx
limit_conn_zone $binary_remote_addr zone=conn_per_ip:10m;
limit_conn conn_per_ip 20;
```

- 限制**同时打开**的连接数，防止单 IP 占满连接或慢速攻击（Slowloris）。
- 与 `limit_req` 分工：`limit_conn` 管并发数，`limit_req` 管速率；通常两个一起配。
- 慢速攻击还需要 `client_header_timeout`、`client_body_timeout`、`send_timeout` 配合收紧。

### 6.3 定时器与超时处理【了解】

Nginx 用红黑树管理定时器，超时事件挂在事件循环上，`epoll_wait` 的超时时间取「最近一个定时器到期时间」。常见超时项：`keepalive_timeout`、`client_header_timeout`、`client_body_timeout`、`send_timeout`、`proxy_connect_timeout`、`proxy_read_timeout`。超时不是直接 RST，而是先走正常的关闭流程（发送剩余数据、按需返回 504 或 408）。另外 `resolver_timeout`、`ssl_handshake_timeout`（1.27+）也属于定时器管理的超时。

### 6.4 高可用架构【重点】

Nginx 自身也会挂，高可用方案有两层：

1. **入口层**：两台 Nginx + keepalived，用 VRRP 虚拟 IP（VIP）做主备，心跳检测失败自动漂移 VIP；或用 LVS/F5/云 SLB 做四层负载，后端挂多台 Nginx。
2. **上游层**：`upstream` 多实例 + 被动健康检查 + `backup`；跨机房再叠加 DNS 轮询 / GSLB。

状态一致性：会话数据放 Redis 等共享存储（不要在 Nginx 存）；配置用配置管理工具统一分发 + 灰度 reload；`keepalived` 脚本要检查 Nginx 进程**和**业务可用性（curl 健康接口），只检查进程会出现「进程在但服务假死」时无法切换。

### 6.5 平滑重载与热升级【重点】

- `nginx -s reload`：master 校验新配置 → 启动新 worker（继承监听套接字）→ 向旧 worker 发 QUIT → 旧 worker 处理完存量连接后退出。**worker 不中断连接**，但正在执行的请求会等它完成，长连接场景要配合 `worker_shutdown_timeout`。
- 二进制热升级：`kill -USR2 <master>` 启动新 master → 向旧 master 发 `WINCH` 停止接收新连接 → 旧 worker 退出后 `QUIT` 旧 master。
- 坑：配置语法错误时 `reload` 失败但旧配置继续生效（要用 `nginx -t` 先校验）；`worker_connections`、`listen` 等参数调整会重启 worker；频繁 reload 会造成内存碎片，建议配合 `worker_shutdown_timeout` 与灰度发布。

### 6.6 日志、错误处理与排障【重点】

| 指令 | 作用 |
| --- | --- |
| `log_format main '$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" rt=$request_time ut=$upstream_response_time';` | 自定义格式，必须记录 `rt` 与 `ut` 定位慢请求 |
| `access_log /var/log/nginx/access.log main buffer=32k flush=1s;` | 缓冲写日志，降低磁盘 I/O |
| `error_log /var/log/nginx/error.log warn;` | 级别 debug/info/notice/warn/error/crit |
| `error_page 500 502 503 504 /50x.html;` + `internal` | 自定义错误页；`internal` 禁止外部直接访问 |
| `error_page 404 =200 /empty.gif;` | `=` 可改写响应码（如统一返回 200 的空白图） |

日志切割：配合 logrotate（按天切割 + `nginx -s USR1` 重新打开文件），或用 `open_log_file_cache` 控制句柄。

**常见错误码定位**：

| 状态码 | 常见原因 |
| --- | --- |
| 499 | 客户端主动断开（上游太慢、用户关页面），看 `upstream_response_time` |
| 502 | 上游拒绝连接/返回非法响应：上游没启动、端口错、上游崩了 |
| 504 | 上游超时：`proxy_read_timeout` 太短或上游慢 |
| 413 | 请求体超过 `client_max_body_size` |
| 400 | 请求头过大（`large_client_header_buffers`）或 Host 非法 |

**内存泄漏排查**：Nginx 本身的内存管理较严谨，泄漏多来自第三方/C 模块。手段：Valgrind（`--leak-check=full`）跑压测、观察 `ps` 的 RSS 随时间增长、审查自定义模块的分配与释放配对、升级到修复版本。worker 内存也可用 `worker_shutdown_timeout` 定期回收（重启 worker）。

## 七、高频面试题与自测清单

### 7.1 高频面试题

| # | 题目 | 答案要点 | 章节 |
| --- | --- | --- | --- |
| 1 | Nginx 与 Apache 的主要区别？ | 事件驱动异步非阻塞 vs prefork 一连接一进程；内存占用低；静态资源强；模块需编译进；Apache 动态处理与模块生态更成熟 | 一 |
| 2 | Nginx 为什么能支撑高并发？ | master + 多 worker，worker 单线程事件循环 + epoll，只返回就绪事件，非阻塞不占线程，单连接内存小 | 一 |
| 3 | 什么是惊群？怎么解决？ | 多 worker 同时 accept 只有一个成功；`accept_mutex` 串行或 `reuseport` 内核分流 | 一 |
| 4 | Nginx 的最大并发怎么估算？ | worker_processes × worker_connections，反向代理减半；还要配 ulimit、somaxconn、端口范围 | 一 |
| 5 | Nginx 的内存是怎么管理的？ | 连接/请求级内存池，整池释放；共享内存用于 SSL 会话、限流状态、upstream zone | 一 |
| 6 | 一个 HTTP 请求在 Nginx 里经过哪些阶段？ | 11 个 phase：POST_READ→SERVER_REWRITE→FIND_CONFIG→REWRITE→POST_REWRITE→PREACCESS→ACCESS→POST_ACCESS→PRECONTENT→CONTENT→LOG | 二 |
| 7 | location 的匹配优先级？ | `=` 精确 → 最长前缀（带 `^~` 直接用）→ 正则按顺序 → 最长前缀兜底 | 二 |
| 8 | root 与 alias 的区别？ | root 拼 URI，alias 替换 location 匹配部分；结尾斜杠要对应 | 二 |
| 9 | server_name 怎么支持多站点？默认 server 是什么？ | 精确→通配→正则→default_server；未匹配走 default_server，否则第一个 server | 二 |
| 10 | try_files 的作用？ | PRECONTENT 阶段按顺序检查文件/目录，都不存在时内部跳转到最后一个 URI（SPA 回退、PHP 404 防护） | 二 |
| 11 | proxy_pass 带斜杠和不带斜杠的区别？ | 带 URI 替换 location 前缀，不带则原样透传 | 三 |
| 12 | Nginx 有哪些负载均衡算法？怎么选？ | 轮询、加权、least_conn、ip_hash、hash consistent、random two；按耗时差异和会话/缓存需求选 | 三 |
| 13 | upstream 的失败重试怎么配？有什么风险？ | max_fails/fail_timeout 被动检查；proxy_next_upstream + tries/timeout；非幂等请求重试会重复提交 | 三 |
| 14 | 为什么要开 proxy_http_version 1.1 和 Connection 空？ | 否则默认 1.0 短连接，upstream keepalive 不生效，TIME_WAIT 暴涨 | 三 |
| 15 | Nginx 的超时参数怎么设？ | connect/send/read 三级；逐层递减，避免前端放弃后端还在算；流式接口放大 read timeout | 三 |
| 16 | 怎么代理 WebSocket？ | `proxy_http_version 1.1` + `Upgrade`/`Connection: upgrade` + 放大 `proxy_read_timeout` | 三 |
| 17 | sendfile 为什么快？loc 怎么配合？ | 内核态直接页缓存到 socket，省两次拷贝；配合 tcp_nopush（凑包）与 tcp_nodelay（低延迟） | 四 |
| 18 | 静态资源有哪些优化手段？ | sendfile、open_file_cache、gzip/gzip_static/brotli、expires 长缓存 + 304、keepalive + HTTP/2、CDN | 四 |
| 19 | gzip 怎么配才不影响性能？ | comp_level 4–6、min_length 1k、只压文本类型、构建期预压缩用 gzip_static | 四 |
| 20 | proxy_cache 怎么防击穿？过期还能返回吗？ | `proxy_cache_lock on` 串行回源；`proxy_cache_use_stale` 上游故障时返回旧缓存 | 四 |
| 21 | HTTPS 怎么配？证书链要注意什么？ | `listen 443 ssl` + cert/key；cert 要含中间证书；80 端口 301 跳转 | 五 |
| 22 | TLS 握手流程？1.2 与 1.3 差异？ | 协商→证书校验→ECDHE→对称加密；1.2 两 RTT，1.3 一 RTT 且只留前向保密套件 | 五 |
| 23 | HTTPS 怎么加固？ | 只启 TLS1.2/1.3、强套件、HSTS、OCSP Stapling、关 session tickets、私钥权限 | 五 |
| 24 | HTTPS 性能怎么优化？ | 会话复用共享缓存、ECDHE+AES-GCM/ChaCha20、OCSP Stapling、ssl_buffer_size、HTTP/2、ECDSA 证书 | 五 |
| 25 | 怎么做 IP 访问控制？顺序有什么坑？ | allow/deny 按书写顺序命中即生效，`deny all` 放最后；可配在 server 或 location | 五 |
| 26 | 还有哪些安全配置？ | server_tokens off、禁危险方法、禁 `.git`/`.env`、autoindex off、安全响应头、WAF | 五 |
| 27 | limit_req 怎么防刷？burst 和 nodelay 是什么？ | 漏桶按 rate 出水，burst 是桶容量，nodelay 让突发立即处理；超限返回 429 | 六 |
| 28 | limit_conn 和 limit_req 的区别？ | 一个限并发连接数，一个限速率；配合超时参数防慢速攻击 | 六 |
| 29 | Nginx 怎么做高可用？ | 双机 + keepalived VIP 或四层 LB；上游多实例 + backup；健康检查要探业务而非仅进程 | 六 |
| 30 | reload 和热升级的原理？ | 新 worker 继承监听套接字、旧 worker 处理完存量连接退出；USR2/WINCH/QUIT 做二进制升级 | 六 |
| 31 | error_page 怎么配？有哪些坑？ | 可映射到本地 URI 或外部地址，`=` 改响应码，`internal` 防外部访问；带 `always` 才在错误码生效 | 六 |
| 32 | 线上出现 502/504/499 怎么排查？ | 502 上游不可用/非法响应；504 上游超时；499 客户端断开；对照 error_log + upstream 时间 | 六 |

### 7.2 动手实验

- [ ] 用 `nginx -t` 校验一份含 `=`、`^~`、`~*` 三类 location 的配置，用 `curl` 逐个验证命中顺序
- [ ] 配置两个 server_name + 一个 default_server，用 `curl -H "Host: xxx"` 验证虚拟主机选择
- [ ] 搭一个 upstream 两台后端，一台 `down`，验证 `proxy_next_upstream` 与 `max_fails` 的摘除与恢复
- [ ] 开启 `keepalive 64` 前后，用 `ss -tan state time-wait | wc -l` 对比 TIME_WAIT 数量
- [ ] 对同一张图片分别关闭/开启 `sendfile`、`gzip`、`open_file_cache`，用 `ab` 或 `wrk` 对比 QPS
- [ ] 配置 `limit_req zone=... burst=5 nodelay`，用循环请求观察 429 的比例
- [ ] 用 `openssl s_client -connect` 检查证书链是否完整、TLS 版本与套件
- [ ] 配置 `proxy_cache` + `add_header X-Cache-Status`，验证 HIT/MISS/STALE

### 7.3 自测清单

- [ ] 能画出一个请求从 accept 到 CONTENT 阶段再到 LOG 阶段的完整路径
- [ ] 能不看资料写出 location 五种修饰符的优先级，并解释 `^~` 的作用
- [ ] 能说清 proxy_pass 有无 URI 的转发差异
- [ ] 能解释为什么 upstream keepalive 不生效，以及怎么修
- [ ] 能说出静态资源加速的至少 5 种手段及各自原理
- [ ] 能讲清 TLS 1.2 与 1.3 的握手差异和加固项
- [ ] 能区分 limit_req 与 limit_conn，并说明 burst/nodelay 的效果
- [ ] 能根据 502/504/499 判断问题大致在哪一层
