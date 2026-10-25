# 构建与 DevOps 核心知识点学习文档

Oct 25, 2026 · @meteorzhong

## 导读

本文展开《知识图谱.html》「构建与 DevOps」分类中的 Maven、Git、CI/CD 三个节点，覆盖依赖管理、版本控制、自动化交付与发布四条主线。标注沿用图谱：**【核心】** 面试必问；**【重点】** 常被追问；**【了解】** 知道概念即可。默认环境为 Maven 3.9 + JDK 17、Git 2.40 以上、Kubernetes 1.28 以上。

| 图谱节点 | 级别 | 本文章节 |
| --- | --- | --- |
| Maven / Gradle（maven） | 重点 | 坐标与仓库、生命周期、依赖 scope 与仲裁、多模块、私服与插件、Maven 与 Gradle 选型 |
| Git（git） | 重点 | 对象模型、merge 与 rebase、reset 与 revert、cherry-pick、分支模型、评审与仓库治理 |
| CI / CD（cicd） | 重点 | 流水线阶段、质量门禁、制品与缓存、配置分离与回滚、蓝绿/滚动/金丝雀 |

**复习顺序**：Maven 坐标与生命周期 → 依赖 scope → 冲突仲裁与排查 → dependencyManagement 与多模块 → 私服与版本语义 → Git 对象模型 → merge/rebase → reset/revert/cherry-pick → 分支模型与 Code Review → CI 流水线与质量门禁 → 制品与构建提速 → 发布策略与数据库兼容 → 优雅上下线与发布观测。面试主线是「依赖冲突怎么定位」「rebase 为什么不能用在公共分支」「发布出问题怎么在分钟级回滚」，三条都要落到具体命令和执行顺序上。

## 一、Maven 与 Gradle 构建

### 1.1 坐标与三类仓库【重点】

Maven 用坐标唯一定位一个构件：`groupId`（组织，反向域名）、`artifactId`（模块名）、`version`（版本）、`packaging`（默认 jar，还有 pom、war）、`classifier`（同版本的可选产物，如 `-sources`、`-javadoc`）。

| 仓库 | 位置 | 说明 |
| --- | --- | --- |
| 本地仓库 | `~/.m2/repository` | 下载后缓存于此，构建优先读本地 |
| 中央仓库 | `repo.maven.apache.org` | 官方公共仓库，只有一个 |
| 私服 | Nexus / Artifactory | 公司内统一入口，代理中央仓库、托管二方包 |

查找顺序是本地 → 私服（`settings.xml` 的 `mirror` 通常拦截 central，所有请求都走私服）→ 中央仓库。私服的价值不只是加速：外网不通时仍能构建、内部构件不外泄、依赖来源可审计、可做安全扫描。

### 1.2 标准目录与生命周期【重点】

| 目录 | 用途 |
| --- | --- |
| `src/main/java` | 主代码 |
| `src/main/resources` | 主资源，打进 jar |
| `src/test/java` | 测试代码，不打包 |
| `src/test/resources` | 测试资源 |
| `target` | 构建输出，可随时 `clean` |

Maven 有三套相互独立的生命周期，执行某个阶段会自动执行它之前的全部阶段：

| 生命周期 | 阶段 | 说明 |
| --- | --- | --- |
| clean | `pre-clean` → `clean` → `post-clean` | 删除 `target` |
| default | `validate` → `compile` → `test` → `package` → `verify` → `install` → `deploy` | 构建主链路，另有 initialize、generate-sources、test-compile 等细分阶段 |
| site | `pre-site` → `site` → `post-site` → `site-deploy` | 生成项目站点文档 |

关键语义：`mvn package` 会依次执行 validate、compile、test、package，产物在 `target/`，**不会**进入本地仓库；`mvn install` 才会装进 `~/.m2`；`mvn deploy` 才推送到私服。生命周期阶段本身不干活，具体动作由插件目标绑定到阶段完成，例如 `maven-compiler-plugin` 绑定 compile、`maven-surefire-plugin` 绑定 test、`maven-jar-plugin` 绑定 package。

### 1.3 依赖范围 scope【重点】

| scope | 编译期 | 测试期 | 运行期 | 打进产物 | 是否传递 |
| --- | --- | --- | --- | --- | --- |
| compile（默认） | 是 | 是 | 是 | 是 | 是 |
| provided | 是 | 是 | 否 | 否 | 否 |
| runtime | 否 | 是 | 是 | 是 | 是 |
| test | 否 | 是 | 否 | 否 | 否 |
| import | 仅用于 `dependencyManagement` 导入 BOM | — | — | — | — |

典型用法：`servlet-api`、`lombok` 用 provided（容器或编译期提供，不能打进包）；JDBC 驱动用 runtime（编译时只需要接口 `javax.sql.DataSource`）；JUnit、Mockito 用 test；`system` 需要写本地绝对路径，破坏可移植性，生产禁用。

### 1.4 依赖冲突仲裁与排查【重点】

Maven 自动引入依赖的依赖，形成依赖树。同一构件出现多个版本时按两条规则选一个：

1. **最短路径优先**：A → B → C(1.0) 与 A → C(2.0)，后者路径更短，选 2.0。
2. **同路径长度先声明优先**：路径一样长时，pom 中先声明的那条胜出。

补充两点常被忽略：`dependencyManagement` 中管理的版本**优先于**传递仲裁；直接显式声明的依赖也优先于传递进来的同构件。被淘汰的版本在树中标记为 `omitted for conflict with x.y.z`。

仲裁只保留一个版本，但被选中的版本未必兼容所有调用方，于是运行期报错：

| 症状 | 含义 |
| --- | --- |
| `NoSuchMethodError` | 编译期存在的方法在运行期版本里没有 |
| `ClassNotFoundException` | 运行时类路径缺类 |
| `NoClassDefFoundError` | 类曾在编译期存在，运行期初始化或加载失败 |
| `AbstractMethodError` / `NoSuchFieldError` | 接口或字段与运行版本不匹配 |

排查与解决：

| 动作 | 命令 / 手段 |
| --- | --- |
| 看完整依赖树 | `mvn dependency:tree -Dverbose` |
| 只看某构件 | `mvn dependency:tree -Dincludes=com.google.guava:guava` |
| 看有效 pom | `mvn help:effective-pom` |
| 看调试日志 | `mvn -X`（或 `-e` 看异常栈） |
| 图形化 | IDEA Maven Helper 插件，直接标红冲突 |
| 构建期拦截 | `maven-enforcer-plugin` 的 `dependencyConvergence` 规则 |

### 1.5 exclusions、dependencyManagement 与 optional【重点】

```xml
<dependency>
  <groupId>com.example</groupId>
  <artifactId>order-api</artifactId>
  <version>1.2.0</version>
  <exclusions>
    <exclusion>
      <groupId>com.google.guava</groupId>
      <artifactId>guava</artifactId>
    </exclusion>
  </exclusions>
</dependency>
```

| 手段 | 作用 | 注意 |
| --- | --- | --- |
| `exclusions` | 在引入方切断某条传递路径 | 精确但有副作用，别人再引入仍会带进来 |
| `dependencyManagement` | 只声明版本，不引入依赖；子模块省略 version | 多模块统一版本的首选 |
| BOM（`import` scope） | 导入一组已管理版本的依赖 | Spring Boot 的 `spring-boot-dependencies` 即此模式 |
| 显式声明 | 直接写所需版本，优先级高于传递 | 简单直接，但版本分散 |
| `optional=true` | 该依赖不传递给下游，用的人自己声明 | 适合驱动、模板引擎等非必需能力 |
| enforcer | 版本不收敛就让构建失败 | 把问题挡在上线前 |

### 1.6 多模块聚合与继承【重点】

| 维度 | 聚合（aggregation） | 继承（inheritance） |
| --- | --- | --- |
| 目的 | 一条命令构建多个模块 | 复用父 pom 的配置 |
| 配置 | 父 pom 的 `<modules>` 列子模块 | 子 pom 的 `<parent>` 指向父 pom |
| 方向 | 父 → 子（构建范围） | 子 → 父（配置来源） |
| 常见误区 | 聚合不会自动带来继承 | 继承不会自动把子模块纳入一次构建 |

父工程 `packaging` 必须是 `pom`。`<relativePath>` 默认 `../pom.xml`，找不到再去仓库找。父 pom 的 `dependencyManagement`、`pluginManagement`、`properties`、`repositories`、`distributionManagement` 会被继承；而父 pom 里直接写的 `<dependencies>` 会被**所有**子模块强制引入，通常不要这么写。Reactor 会按模块间依赖关系计算构建顺序，被依赖的模块先构建。

### 1.7 私服 Nexus 与 SNAPSHOT/RELEASE【重点】

Nexus 有三类仓库：`proxy` 代理中央或第三方仓库并缓存；`hosted` 托管公司构件，分 `releases` 与 `snapshots`；`group` 把多个仓库聚合成一个对外 URL。`settings.xml` 里配置 `mirror` 指向 group，`server` 配置发布凭证，项目 pom 用 `distributionManagement` 指定 `repository` 与 `snapshotRepository`。

| 维度 | SNAPSHOT | RELEASE |
| --- | --- | --- |
| 版本号 | `1.2.0-SNAPSHOT` | `1.2.0` |
| 可否重复发布 | 可以，私服按时间戳保留多份 | 不可以，同版本号覆盖会失败 |
| 依赖方行为 | 每次构建检查最新快照，`mvn -U` 强制更新 | 首次下载后不再变化 |
| 适用阶段 | 联调、频繁迭代的二方包 | 测试、预发、生产 |
| 风险 | 同一版本号内容可能不同，构建不可复现 | 内容固定，可复现 |

规范是生产环境**禁止依赖 SNAPSHOT**，否则同一份代码在不同时间构建出的产物可能不同，出了问题无法复现。发布 release 前要确保快照依赖已全部转正。

### 1.8 常用插件与 Maven/Gradle 选型【了解】

| 插件 | 作用 |
| --- | --- |
| `maven-compiler-plugin` | 编译，配置 `release`（推荐）或 source/target |
| `maven-surefire-plugin` | 跑单元测试（`*Test`）与集成测试（`*IT`，由 failsafe 绑定 verify） |
| `maven-shade-plugin` | 打 uber jar，支持 `relocation` 重定位包名解决冲突 |
| `maven-assembly-plugin` | 自定义分发包（zip/tar，含脚本与配置） |
| `spring-boot-maven-plugin` | `repackage` 生成可执行 jar，`layered` 支持镜像分层 |
| `jacoco-maven-plugin` | 覆盖率报告，供 CI 门禁读取 |

| 维度 | Maven | Gradle |
| --- | --- | --- |
| 配置方式 | XML，约定优于配置，结构统一 | Groovy / Kotlin DSL，可写逻辑 |
| 构建性能 | 全量构建为主，大项目偏慢 | 增量构建、构建缓存、守护进程，通常更快 |
| 冲突策略 | 最短路径 + 先声明 | 默认选最高版本，可用 `force` / `strictly` 锁定 |
| 学习成本 | 低，生态最普及 | 高，需理解 DSL 与任务模型 |
| 适用 | 团队大、需要标准化的后端服务 | 多模块复杂构建、对构建速度敏感、Android 与 Spring 官方项目 |

## 二、Git

### 2.1 三区、对象模型与引用【重点】

| 区域 | 位置 | 说明 |
| --- | --- | --- |
| 工作区 | 磁盘目录 | 正在编辑的文件 |
| 暂存区（index） | `.git/index` | `git add` 后的内容，下一次提交的快照 |
| 本地仓库 | `.git/objects` | `git commit` 后的提交历史 |
| 远程仓库 | origin 等 | 协作与备份 |

Git 是内容寻址的对象数据库：`blob` 存文件内容，`tree` 存目录与文件名到 blob/tree 的映射，`commit` 存一个 tree、父提交、作者与提交信息，`tag`（附注标签）指向某个提交。对象以内容哈希命名，内容不变哈希不变，所以提交一旦生成就不可修改——所谓「修改提交」其实是新建对象。

**分支只是一个指向提交的指针**（`.git/refs/heads/` 下存 40 字节 SHA），因此创建、切换分支几乎零成本。`HEAD` 指向当前分支；`git checkout <commit>` 进入 detached HEAD，此时提交不属于任何分支，切走就可能丢，需要新建分支接住。`git log --graph --oneline` 和 `git cat-file -p <sha>` 是理解历史结构最直接的工具。

### 2.2 merge 与 rebase【重点】

| 维度 | merge | rebase |
| --- | --- | --- |
| 原理 | 以共同祖先做三方合并，生成两个父节点的合并提交 | 把你的提交逐个重放到目标分支最新提交之后 |
| 历史 | 保留真实分叉，图较乱 | 线性整洁 |
| commit ID | 原有提交不变，新增合并提交 | 全部重写，生成新 ID |
| 冲突 | 一次解决 | 可能每个提交都要解决 |
| 适用 | 合入主干、公共分支 | 本地整理、同步主干 |

`git merge` 在目标分支可以直接前移时默认 fast-forward，不产生合并提交；`--no-ff` 强制生成合并提交，保留「这里曾有一个功能分支」的边界，便于整体 revert 一个功能。`git rebase -i HEAD~3` 可 `pick`、`reword`、`squash`、`fixup`、`drop`，把零碎提交整理成有意义的几个。

**铁律：不要 rebase 已推送的公共分支。** 重写后提交 ID 全变，必须强制推送，其他人基于旧提交的工作会对不上，再合并会出现重复提交和大量冲突。本地功能分支同步主干用 `git pull --rebase`；合并请求合并回主干用 merge 或 squash merge。必须强推时用 `git push --force-with-lease`，它会检查远端是否有别人的新提交，比 `--force` 安全。

### 2.3 reset 三种模式与 revert【重点】

| 模式 | 分支指针 | 暂存区 | 工作区 | 典型用途 |
| --- | --- | --- | --- | --- |
| `--soft` | 移动 | 保留 | 保留 | 撤掉提交但保留改动在暂存区，重新组织提交 |
| `--mixed`（默认） | 移动 | 重置 | 保留 | 取消暂存，改动留在工作区 |
| `--hard` | 移动 | 重置 | 重置 | 彻底丢弃，危险，未提交改动无法找回 |

`reset` 改写历史，只适合本地未推送的提交。`revert` 是新增一个内容相反的提交来抵消指定提交，历史不改、记录可追溯，适合已推送的公共分支与线上主干；日后还可以 revert 这次 revert 把功能找回来。回滚合并提交要用 `git revert -m 1 <merge-sha>`，`-m 1` 表示保留第一个父（通常是目标分支）为主线。误操作后可查 `git reflog`，用 `git reset --hard HEAD@{n}` 或先建 `backup` 分支接住丢失的提交。

### 2.4 cherry-pick 与 stash【重点】

`cherry-pick` 把指定提交的补丁复制到当前分支，生成内容相同、ID 不同的新提交。典型场景：线上 hotfix 提交在开发分支，只把这一个修复摘到 release 分支；多个维护版本同步同一个补丁。连续摘取用 `git cherry-pick -x` 记录来源；冲突按普通冲突处理；频繁 cherry-pick 会产生同内容不同 ID 的提交，后续合并可能重复冲突。

| 命令 | 作用 |
| --- | --- |
| `git stash push -m "说明"` | 保存工作区与暂存区改动 |
| `git stash -u` | 连未跟踪文件一起保存 |
| `git stash list` | 查看 stash 栈 |
| `git stash pop` / `apply` | 恢复并删除 / 恢复但保留 |

stash 是临时的，不是备份；切分支前保存改动可以避免「切换失败」或把无关改动带进提交。

### 2.5 冲突解决流程【重点】

1. 先 `git fetch`，看清本地与远端差异，不要盲目合并。
2. merge 或 rebase 冲突后 `git status` 会标出 `both modified`，编辑文件处理 `<<<<<<<`、`=======`、`>>>>>>>` 三段。
3. 先理解双方意图再动手，不确定就找对方确认；不要简单选一边。
4. `git add` 标记已解决，`git merge --continue` 或 `git rebase --continue`；想放弃用 `--abort`。
5. 解决后必须重新编译并跑测试，冲突可能编译通过但语义错误。
6. 注意 ours/theirs 的语义在 rebase 下是反的：rebase 时 `ours` 是目标分支，`theirs` 是你的提交。

减少冲突的实践：分支寿命短、频繁同步主干、PR 尽量小、模块边界清晰、统一格式化工具（避免整文件重排）、必要时用 `.gitattributes` 指定合并策略。

### 2.6 Git Flow 与 Trunk Based Development【重点】

| 维度 | Git Flow | Trunk Based |
| --- | --- | --- |
| 长期分支 | `master` + `develop` | 只有主干（`main`） |
| 短期分支 | `feature/*`、`release/*`、`hotfix/*` | 存活 1 到 2 天的短分支 |
| 发布节奏 | 固定周期、多版本并行维护 | 持续集成、随时可发布 |
| 分支寿命 | 长，合并冲突多 | 短，冲突少 |
| 前置条件 | 流程纪律、版本管理 | 完善自动化测试、CI、功能开关 |
| 适用 | 客户端软件、需要同时维护多个版本的产品 | 互联网服务、SaaS、持续部署团队 |

Git Flow 的 `release` 分支做测试与修复，测完同时合入 `master` 与 `develop` 并打 tag；`hotfix` 从 `master` 拉出，修完同样合回两边。Trunk Based 要求未完成功能用功能开关隐藏，主干始终绿、始终可发布，Code Review 通过短小的 PR 完成。评审配合方式：Git Flow 的评审发生在 feature → develop、release → master；Trunk Based 的评审发生在每次小 PR 合入主干，要求更快、更小、更频繁。

### 2.7 PR/MR 评审要点【重点】

| 要点 | 做法 |
| --- | --- |
| 体量 | 控制在几百行以内，大需求拆多个 PR |
| 描述 | 背景、方案、影响面、测试方式、回滚方案 |
| 准入 | CI 全绿、至少一人 approve、无未解决讨论 |
| 关注点 | 设计合理性、正确性、边界与异常、并发、安全、兼容性、可观测性 |
| 不关注 | 代码风格交给 formatter / linter，人工不纠缠格式 |
| 合并策略 | 团队统一 merge、squash 或 rebase，避免历史形态混杂 |

评审意见要具体、可执行，区分「必须改」与「建议」；作者及时响应，不把 PR 拖成长寿分支。

### 2.8 仓库体积、LFS 与敏感信息应急【重点】

二进制大文件（模型、视频、设计稿、超大 jar）直接入库会让每次 clone 都下载全部历史。用 Git LFS 把大文件替换为指针文件，真实内容存 LFS 服务器；历史已经膨胀的，用 `git filter-repo`（推荐）或 BFG 清理后强推，并通知所有人重新克隆。日常可用浅克隆 `--depth 1` 加速 CI。

敏感信息（密钥、密码、Token、证书）误提交的处置顺序：

1. **立即吊销并轮换凭证**，这是第一优先级，清理历史不能挽回已经泄露的内容。
2. 用 `git filter-repo` / BFG 从全部历史中删除该文件或字符串。
3. 强制推送并通知所有协作者重新克隆，旧克隆里仍有残留。
4. 加 `pre-commit` 钩子（gitleaks、detect-secrets）与 CI 密钥扫描，补 `.gitignore`。
5. 检查 fork、PR、CI 日志、镜像层等副本，确认没有二次留存。

## 三、CI / CD

### 3.1 流水线阶段【重点】

| 阶段 | 做什么 | 失败处理 |
| --- | --- | --- |
| 拉代码 | 浅克隆，恢复依赖缓存 | 重试并告警 |
| 编译 | `mvn -B -DskipTests package` | 阻断 |
| 单元测试 | surefire + JaCoCo 覆盖率 | 阻断 |
| 静态检查 | SonarQube、Checkstyle、SpotBugs | 阻断（按门禁） |
| 依赖与镜像扫描 | OWASP Dependency-Check、Trivy | 高危阻断 |
| 构建镜像 | Docker 多阶段构建，tag 用 commit SHA | 阻断 |
| 推送仓库 | Harbor / ECR，记录 digest | 重试 |
| 部署测试环境 | 自动部署 + 接口/冒烟测试 | 阻断 |
| 预发与审批 | 人工审批、生产变更单 | — |
| 生产发布 | 滚动 / 蓝绿 / 金丝雀 + 观测 | 自动或一键回滚 |

流水线由 push、MR、tag 或定时触发。流水线即代码：GitLab CI 用仓库根目录 `.gitlab-ci.yml`，Jenkins 用 `Jenkinsfile`，两者都与代码同版本管理、可评审、可复用。

### 3.2 Jenkins 与 GitLab CI【重点】

| 维度 | Jenkins | GitLab CI |
| --- | --- | --- |
| 形态 | 自建服务，插件生态庞大 | 与代码仓库一体，开箱即用 |
| 配置 | `Jenkinsfile`（声明式 / 脚本式） | `.gitlab-ci.yml` |
| 执行节点 | master + agent，标签匹配 | Runner，按 tag 与并发选择 |
| 缓存 | 插件或挂载 workspace | `cache`（按 key 复用目录）+ `artifacts`（阶段间传递） |
| 维护成本 | 高，插件与版本兼容需治理 | 低，升级随平台 |
| 适用 | 历史包袱重、需要复杂编排与多系统集成 | 已用 GitLab、希望快速落地 |

```yaml
stages: [build, test]
cache:
  key: maven-$CI_COMMIT_REF_SLUG
  paths: [.m2/repository]
build:
  stage: build
  script: [mvn -B -DskipTests package]
  artifacts: {paths: [target/*.jar], expire_in: 1 day}
```

缓存 key 要绑定锁文件或分支，避免不同分支互相污染；缓存只是加速，**不能作为正确性依赖**，缓存丢失时流水线必须仍能成功。

### 3.3 质量门禁与 SonarQube【重点】

| 门禁项 | 常见阈值 | 工具 |
| --- | --- | --- |
| 编译 | 零错误，告警纳入统计 | javac、-Werror（谨慎） |
| 单元测试 | 通过率 100% | surefire |
| 新增代码覆盖率 | 60% 到 80% | JaCoCo + SonarQube |
| 静态问题 | 新增代码 0 blocker / 0 critical | SonarQube Quality Gate |
| 重复率 | 新增代码不超过 3% | SonarQube |
| 依赖漏洞 | 无 critical / high，或有审批豁免 | Dependency-Check、Trivy |

门禁要**卡新增代码**而不是全量存量，否则历史债务会让门禁永远无法开启。豁免必须有审批人、理由和到期时间，不能一键永久跳过。

### 3.4 制品版本与不可变镜像【重点】

原则是**一次构建、多处部署**：同一份镜像从测试、预发到生产逐级晋级，绝不在每个环境重新构建。tag 使用 `git commit SHA`（短 SHA）或语义版本，同时记录镜像 digest；禁止用 `latest` 部署，因为它不可追溯、无法确定实际运行的是哪次构建。同一 tag 不允许覆盖推送，Harbor 可开启 immutable 策略。

可追溯链应为：commit → 流水线编号 → 制品/镜像 digest → 部署记录 → 监控指标按版本打标签。制品仓库设置保留策略，长期版本与最近若干次构建保留，其余归档。

### 3.5 构建缓存与提速【重点】

| 手段 | 说明 |
| --- | --- |
| 依赖缓存 | 缓存 `~/.m2/repository`、Gradle cache，按锁文件或分支设 key |
| Docker 分层缓存 | 先 `COPY pom.xml` 再 `dependency:go-offline`，最后 `COPY src`，改代码不重下依赖 |
| 多阶段构建 | 构建阶段与运行阶段分离，运行镜像只含 JRE 与 jar |
| 并行 | Maven `-T 1C`、Gradle parallel、测试分片、独立 stage 并行 |
| 增量构建 | 只构建受影响模块，Gradle 增量与 build cache 效果明显 |
| 自托管 Runner | 靠近制品仓库与集群，减少拉取时间；大项目给足 CPU 与内存 |

提速的前提是不牺牲可复现性：缓存可失效、构建结果只依赖代码与锁定的依赖版本。

### 3.6 环境与配置分离、回滚【重点】

配置遵循十二要素：镜像内不写环境配置，通过环境变量或配置中心（Nacos、Apollo）注入；密钥放 K8s Secret 或 Vault，不进代码库；启动时校验必需配置，缺失即 fail fast；配置变更同样要版本化、审计、可回滚。

| 回滚方式 | 说明 | 速度 |
| --- | --- | --- |
| 重新部署上一镜像 | 用上一版本 tag/digest 重新发布 | 分钟级 |
| K8s `rollout undo` | 回到上一个 ReplicaSet | 分钟级 |
| 蓝绿切回 | 流量切回旧环境 | 秒级 |
| 金丝雀归零 | 新版本流量降为 0 | 秒级 |
| 功能开关关闭 | 代码已发布但功能不生效 | 秒级 |

数据库变更通常**不随代码回滚**，因此发布前必须保证 schema 前向兼容，回滚代码后旧代码仍能读写新结构。回滚要演练，不能等出事才第一次执行。

## 四、发布与部署策略

### 4.1 蓝绿、滚动、金丝雀【重点】

| 维度 | 滚动 | 蓝绿 | 金丝雀 |
| --- | --- | --- | --- |
| 做法 | 逐批替换旧实例 | 两套环境，验证后一次性切流 | 少量流量引到新版本，观察后放量 |
| 额外资源 | 不需要 | 约双倍 | 少量 |
| 回滚速度 | 较慢，逐批回滚 | 秒级切回 | 秒级归零 |
| 影响面 | 发布期间新旧共存 | 新版本有问题则全量受影响 | 仅少量用户 |
| 依赖 | K8s Deployment 默认 | 负载均衡 / 网关 | 网关、服务网格、功能开关 |
| 适用 | 常规迭代 | 关键系统、要求快速回滚 | 重大变更、新功能验证 |

滚动发布对应 K8s 的 `maxUnavailable` 与 `maxSurge`，要保证可用副本数不低于下限；金丝雀一般按 1% → 5% → 25% → 50% → 100% 阶梯放量，每步观察错误率、延迟与核心业务指标，并设自动回滚条件。

### 4.2 数据库变更与代码发布的兼容顺序【重点】

采用 expand-contract（扩展-收缩）模式，顺序固定：

1. **扩展**：先做兼容性 DDL，只加不删，新字段可空或带默认值，新表新建，旧代码无感知。
2. **迁移**：回填历史数据，双写新旧字段，代码同时兼容新旧结构。
3. **发布代码**：新代码读写新结构，此时新旧实例可能同时在线，必须双向兼容。
4. **收缩**：观察稳定后，再删除旧字段、旧索引、旧代码路径。

规则：DDL 与 DML 分开执行；大表变更用 `gh-ost` 或 `pt-online-schema-change` 避免锁表；改字段类型分多步（加新列 → 双写 → 切读 → 删旧列）；不可逆变更先备份并准备补偿脚本；索引尽量在线创建。回滚代码时不回滚 schema，所以任何一步都要保证上一版本代码仍能运行。

### 4.3 Java 服务优雅上下线【重点】

| 阶段 | 动作 | 目的 |
| --- | --- | --- |
| 启动 | readiness 探针通过后才接流量，必要时 warmup 预热 | 避免把请求打到未就绪实例 |
| 下线第一步 | preStop `sleep 5` 到 `15` 秒 | 等 Endpoints 变更传播到 kube-proxy / Ingress |
| 下线第二步 | readiness 置为失败 | 从 Service 摘除，不再接新请求 |
| 下线第三步 | 收到 SIGTERM，停止接收新请求 | 进入停机流程 |
| 下线第四步 | 等待在途请求完成，关闭线程池、连接池、MQ 消费者 | 不丢请求、不丢消息 |
| 下线第五步 | 注销注册中心实例，进程退出 | 调用方不再发现该实例 |

Spring Boot 配置 `server.shutdown=graceful` 与 `spring.lifecycle.timeout-per-shutdown-phase=30s`；`terminationGracePeriodSeconds` 必须大于 preStop 时间加停机时间，否则会被 SIGKILL。注册中心（Nacos、Eureka）要先下线再停进程，MQ 消费者要先停止拉取再关闭连接。

### 4.4 发布观测【重点】

| 指标 | 关注点 |
| --- | --- |
| 错误率 | 5xx 比例、异常数量，按版本与实例对比 |
| 延迟 | TP99 / TP999，而不是只看平均值 |
| 业务指标 | QPS、下单成功率、支付成功率、核心接口成功率 |
| 资源 | JVM GC、堆、线程池队列、CPU、数据库连接池 |
| 依赖 | 下游错误率、超时率、熔断状态 |

观测要按版本打标签，才能对比新旧差异；观察窗口至少覆盖一个业务波峰；告警阈值与自动回滚条件提前配置；日志与 Trace 中带上版本号，便于定位问题实例。发布期间禁止无关变更，避免多因素干扰判断。

## 五、高频面试题与自测清单

### 5.1 高频面试题

| # | 题目 | 答案要点 | 章节 |
| --- | --- | --- | --- |
| 1 | Maven 依赖传递受什么影响？ | 自动引入依赖的依赖；scope 决定传递，test、provided 不传递；optional 不传递 | 1.3、1.5 |
| 2 | 依赖冲突的仲裁规则？ | 最短路径优先；同长度先声明优先；dependencyManagement 与直接声明优先于传递 | 1.4 |
| 3 | 依赖冲突的典型症状？ | NoSuchMethodError、ClassNotFoundException、NoClassDefFoundError、AbstractMethodError | 1.4 |
| 4 | 怎么排查依赖冲突？ | `mvn dependency:tree -Dverbose -Dincludes`、Maven Helper、`help:effective-pom`、`mvn -X` | 1.4 |
| 5 | 依赖冲突怎么解决？ | exclusions 切断路径；dependencyManagement / BOM 统一版本；显式声明；enforcer 收敛；shade relocation | 1.5 |
| 6 | 聚合和继承的区别？ | modules 为了一起构建；parent 为了复用配置；聚合不带来继承，父 pom packaging=pom | 1.6 |
| 7 | Nexus 和 SNAPSHOT/RELEASE 的作用？ | 代理缓存、托管二方包、统一管控；SNAPSHOT 可覆盖用于联调，release 不可变；生产禁用 SNAPSHOT | 1.7 |
| 8 | Maven 和 Gradle 怎么选？ | Maven 标准、易上手、生态广；Gradle 灵活、增量与缓存快、适合复杂大项目 | 1.8 |
| 9 | merge 和 rebase 的区别？ | merge 生成双父合并提交保留真实历史；rebase 重放提交、历史线性但改写 commit ID | 2.2 |
| 10 | 什么时候不能用 rebase？ | 已推送的公共分支；会导致他人历史不一致；强推用 `--force-with-lease` | 2.2 |
| 11 | reset 三种模式和 revert 怎么选？ | soft/mixed/hard 分别影响指针、暂存区、工作区；公共分支用 revert，本地未推送用 reset | 2.3 |
| 12 | cherry-pick 用在什么场景？ | hotfix 摘到 release、多版本同步补丁；生成新 ID；注意重复提交与冲突 | 2.4 |
| 13 | Git Flow 和 Trunk Based 的区别？ | 长期双分支 + feature/release/hotfix vs 短分支频繁合入主干 + 功能开关；后者依赖 CI | 2.6 |
| 14 | Code Review 和冲突解决怎么做？ | 小 PR、至少一人审核、CI 通过；关注设计边界安全；冲突时理解双方意图、解决后跑测试 | 2.5、2.7 |
| 15 | CI/CD 流水线和质量门禁怎么设置？ | 编译、单测覆盖率、SonarQube、依赖漏洞；卡新增代码；镜像 tag 用 commit SHA；逐级晋级 | 3.1、3.3 |
| 16 | 蓝绿、滚动、金丝雀的区别与选型？ | 资源、回滚速度、影响面、路由依赖不同；常规滚动、关键系统蓝绿、重大变更金丝雀 | 4.1 |
| 17 | 数据库变更和代码发布怎么排顺序？ | expand-contract：先兼容扩展、回填双写、发布代码、观察后收缩删除；回滚代码不回滚 schema | 4.2 |
| 18 | Java 服务怎么优雅上下线？ | readiness 接流量；preStop 等待摘除；SIGTERM 后 graceful shutdown；停线程池、连接池、MQ | 4.3 |
| 19 | 敏感信息误提交怎么处理？ | 先吊销轮换凭证；filter-repo/BFG 清历史；强推并重新克隆；pre-commit 与 CI 扫描 | 2.8 |

### 5.2 动手实验

- [ ] 在项目中引入两个路径不同的 Guava 版本，用 `dependency:tree -Dverbose` 找出被 omitted 的版本
- [ ] 用 `dependencyManagement` 统一多模块版本，再用 enforcer 的 `dependencyConvergence` 验证构建是否失败
- [ ] 搭一个父子多模块工程，分别验证「只聚合不继承」和「只继承不聚合」的效果
- [ ] 在本地分支做一次 rebase，用 `git reflog` 对比 rebase 前后的 commit ID
- [ ] 分别用 `reset --soft/--mixed/--hard` 撤回同一提交，观察 index 与工作区差异
- [ ] 模拟一次合并提交回滚，执行 `git revert -m 1` 并验证历史未被改写

### 5.3 自测清单

- [ ] 能说清 Maven 三类仓库的查找顺序，以及 `package`、`install`、`deploy` 的差别
- [ ] 能默写五种 scope 的编译、运行、打包与传递行为
- [ ] 能独立排查一次 `NoSuchMethodError`，并给出三种解决方案
- [ ] 能解释聚合与继承的区别，以及父 pom 中 `dependencies` 与 `dependencyManagement` 的差异
- [ ] 能说清 merge 与 rebase 的底层差异，并解释为什么公共分支禁止 rebase
- [ ] 能根据场景选择 reset、revert、cherry-pick，并说明风险
- [ ] 能对比 Git Flow 与 Trunk Based，并说明各自对 CI 和评审的要求
- [ ] 能设计一条从提交到生产的流水线，包含质量门禁、制品版本与回滚方案
- [ ] 能说清蓝绿、滚动、金丝雀的资源、回滚速度与适用场景
- [ ] 能描述一次数据库字段变更的完整发布顺序与回滚边界
