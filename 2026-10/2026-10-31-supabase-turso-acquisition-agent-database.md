# Supabase 收购 Turso 深度拆解：当数据库的创建单位从"公司"变成"Agent"

> 数据截至 2026-10-31。文中涉及的 Supabase / Turso 运营数据与性能数字均为厂商口径，未经独立复核；第三方评测与社区反馈已单独标注。

## 一、事件：同一天的两张牌

2026 年 10 月 2 日，在 Supabase Select 2026 大会上，Supabase 同时宣布了两件事：

1. **完成 1.5 亿美元新融资**，由新加坡主权基金 GIC 领投，Alphabet 旗下 CapitalG、IronArc、SquarePeg 跟投；
2. **收购 Turso**，价格未披露。

先把一个常见的误称校准掉：Supabase 收购的不是 SQLite——SQLite 是开源共同体项目，无公司可收购。Turso 是一家把 SQLite 用 Rust 从头重写了一遍的数据库创业公司（重写版代号 Limbo，早期产品为 libSQL fork）。收购公告中，Turso 联合创始人 Glauber Costa 出任 Supabase「Head of Agentic Services」，另一位联合创始人 Pekka Enberg 与全团队一并加入；Turso 平台继续运营，Turso Database 保持开源。

融资本身也值得放在坐标系里看：四个月前（2026 年 6 月），Supabase 刚完成 5 亿美元 F 轮、估值 105 亿美元，领投方同样是 GIC。也就是说，**2026 年一年内 Supabase 累计融资 6.5 亿美元**——这不是常规节奏，是战略抢跑的节奏。

## 二、Supabase 为什么要买 Turso：三层动机

### 2.1 经济学层：Agent 把"建库"变成了高频操作

这笔交易的动机，藏在 Supabase 自己披露的数字里：

| 指标 | 口径 | 来源 |
|---|---|---|
| 每周新建数据库 | 超过 100 万个 | Supabase 官方博客 |
| 每月新增用户 / 数据库 | 100 万+ 用户、400 万+ 数据库 | PR Newswire 融资稿 |
| 新库中由 AI 工具创建的比例 | 约 70% | 同上 |
| 数据库创建量同比增幅 | 600%（截至 2026 年 6 月的一年） | F 轮披露 |
| 最大新增用量来源 | Claude Code（2026 年以来） | F 轮披露 |

注意最后一行：**平台最大的"用户"已经不是人，而是 Agent。**人建库的习惯是"一建管好几年"，Agent 建库的习惯是"为一个原型、一次探索、一个临时看板各建一个，用完即弃"。当创建单位从"每公司一个库"碎化成"每任务一个库"，传统的"每个库背后一台常驻机器"的成本模型直接失效——要么太贵，要么唤醒太慢。

Turso 的客户案例把这笔账算得很白：Engine Labs 的 CTO.new 产品给每个项目配一个 Turso 库做多 Agent 协调层，跑着数万个数据库、服务 5 万多用户。它原来用 Supabase，每用户每月约 20 美元；迁到 Turso 后边际数据库成本从 20 美元降到约 1 美元，整体降约 95%，月费套餐 500 美元（Turso 官方案例口径）。**这就是"每 Agent 一库"经济模型成立的前提：建库成本趋近于零。**

### 2.2 入口层：争夺 Agent 的"第一个数据库"

Supabase CEO Paul Copplestone 的原话：

> "Agents are spinning up millions of databases to power the prototypes, explorations, dashboards, and apps they're building. This new pattern, at this massive scale, requires an evolution in database infrastructure."

翻译过来：Agent 正在成为技术的主要用户，而 **谁定义了 Agent 第一次建库的体验，谁就大概率拿到它后端的第一个入口**。SQLite 是那扇门，Postgres 是门后的主库，中间的 Auth、Storage、Edge Functions、集成生态和 MCP 是把用户留在院子里的篱笆。

Turso 创始人 Glauber Costa 的目标说得更露骨：服务十亿个数据库。他加入 Supabase 后还提出一个可扩展性的新维度——以前看"能存多少数据、查询多快"，现在要加第三问：**能多快配置、管理多少个数据库**。

### 2.3 技术栈层：买下 Postgres 的"下限"

Supabase 自己的叙事很克制：Postgres 继续做，Turso 继续做，SQLite 管小而按需的负载，Postgres 管规模化之后的负载。同一批公布的还有两个自研件：

- **Multigres**：出自 Vitess 团队的开源 Postgres"操作系统"，同步复制之上加共识协议，节点故障秒级切换、已提交写入不丢，应用不改代码；v0.1 alpha；
- **OrioleDB**：换掉 Postgres 老存储引擎，undo log 取代 heap，顺带解决表膨胀、VACUUM 开销、32 位 XID 回卷，官方 TPC-C 衍生基准吞吐最高提升 1.8 倍。

拼起来是一张完整的"毕业路径"图：

```
Agent（每会话/每任务/每用户一个库，按需创建、用完即弃）
  → Turso · SQLite：单文件库、无冷启动、闲置不耗算力、MVCC 并发写
  → Supabase · Postgres：Auth / Storage / Edge Functions / MCP
  → Multigres + OrioleDB：水平扩展、自动故障转移、PB 级
```

**收购买的是 Postgres"上限"与 SQLite"下限"之间的整条带宽，以及"数据库数量"这个新的可扩展性维度。**这解释了为什么是"收购"而不是"自研"：diskless 架构、MVCC 重写、百万库单节点的工程积累，不是砸钱一年能追平的。

## 三、Turso 到底做对了什么：把数据库变回一个文件

Turso 前身是 2021 年成立的 ChiselStrike。创始团队是系统底层出身（Glauber Costa 有 Linux 内核、ScyllaDB、Datadog 约 25 年背景）。技术路线分两步：

1. **libSQL 阶段（2022–2024）**：fork SQLite 加远程复制和向量搜索。但很快就撞墙：SQLite 官方测试套件是专有的、代码是 C 的，想大改心里没底；
2. **Limbo 阶段（2024 年 12 月启动）**：用 Rust 把 SQLite 从零重写，SQL 方言、文件格式、C API 完全兼容，内核内存安全、现代化。2026 年 3 月发布首个支持并发写、CDC、原生向量搜索的版本。

真正的工程壁垒在"盘"这一层，Turso 称之为 **diskless**：

- 计算节点完全无状态，本地 NVMe 只当读缓存，**S3 Express One Zone 是所有已提交事务的唯一真相来源**；
- 写路径：事务先在内存跑完，等几毫秒把同一节点上多个库的 WAL 合并成一次 S3 PUT，成功应答后再回执客户端；
- 压测口径（4KB 操作、同可用区）：S3 Express PUT 平均 6.4ms，本地 fsync 约 2ms——小对象上只多花约 4ms；标准 S3 则要 31ms。

这套组合拳换来三个别人很难同时给的结果：

| 能力 | 机制 | 量化口径（厂商） |
|---|---|---|
| 单节点百万库 | 库=文件，不是进程 | 无冷启动，闲置仅占存储 |
| 空闲零算力 | "没有什么可以 scale to zero" | 唤醒无惩罚 |
| 单库低成本 | 多库 WAL 合并 PUT | 约 0.57 美元/库/月 |

再加一个关键补丁——**BEGIN CONCURRENT**：受 SQL Server Hekaton 启发的 MVCC，写事务并发跑、只在提交时对双方都写过的行做冲突检测，解决 SQLite 单写入者（全局写锁、SQLITE_BUSY）的老毛病。官方实测口径：每事务 1ms 计算、8 线程下写吞吐约为原生 SQLite 的 4 倍。

还有两个容易被忽略的细节：文件切成 128kB segment，一组 segment 构成 generation，**分支只是一次元数据操作，几乎瞬时完成**——这是"每个 Agent 一个库、还能随便复制"的底层原因；支持 BYOC（整套跑在客户自己 AWS 账户）和 BYOK（每库独立密钥、对 Turso 不透明）。

边界也必须说清：**MVCC 当时仍是 tech preview**（不支持 CREATE INDEX，行版本内存开销不小）；Rust 重写版官方自称"尚未到 1.0"；Hacker News 社区有人复测指出 Turso 在 ClickBench 上慢于原生 SQLite。它是为"海量小库"设计的专用武器，不是通用替代品——数据集超过数 GB、高并发多进程写、复杂分析查询，第三方评测的结论都是回到 Postgres。

## 四、第二问：为什么 Agent 一定要用数据库

这是比收购本身更根本的问题。结论先行：**不是"Agent 选择了数据库"，而是"Agent 要可靠地干活，就绕不开数据库提供的四样东西"——持久记忆、事务性状态、结构化检索、隔离边界。**

### 4.1 LLM 在 API 层面是无状态的

大模型本身不记任何事：响应完就忘。Agent 要跨会话工作、要从经验中学习、要执行多步任务，就必须把"记忆"外置到某种持久存储里。业内通常把 Agent 的记忆拆成四层：

| 记忆层 | 内容 | 对存储的要求 |
|---|---|---|
| 短期记忆 | 当前会话上下文 | 低延迟读写，内存/缓存级 |
| 情景记忆 | 历史会话、事件日志 | 结构化存储 + 时间序检索 |
| 语义记忆 | 知识嵌入、RAG | 向量索引 + 混合检索 |
| 程序性记忆 | 工作流、指令、技能状态 | 事务性结构化存储 |

"向量数据库 alone" 是常见误区：向量库解决语义检索，但不解决 ACID 状态转移、多步工作流、结构化 join。生产级 Agent 的真实负载是 SQL 数据 + 嵌入 + 事件日志 + 工作流状态的混合物。

### 4.2 状态机需要事务，任务会中断、会重试

Agent 执行长任务是状态机：读状态 → 决策 → 写状态 → 下一步。任务可能中途崩溃、可能被重试、可能需要回滚。如果没有 ACID 保证，一次部分写入就会把 Agent 卡进不一致的死角。文件系统能存数据，但给不了"要么全成、要么全不成"的原子承诺；纯内存更不行——进程一死全丢。

### 4.3 多 Agent 需要隔离，隔离的自然单位是"库"

多 Agent 系统引入"租户 × Agent × 分支"的爆炸式组合：每个 Agent 要有自己的记忆、不能互相污染状态、要能做并行实验和安全回滚。**把一个 Agent 的数据放进独立的库，是最便宜、最清晰的隔离边界**——比在一套共享表上加行级安全策略简单一个数量级。Turso 的瞬时分支（generation 元数据操作）本质上是把"给 Agent 开隔离环境"做成了 O(1) 操作。

### 4.4 成本拐点：当库便宜到可以"用完即弃"

前面 CTO.new 的案例是这条的实证：边际成本 20 美元 → 1 美元，租户的隔离粒度才可以从"逻辑行"放宽到"整库"。**经济学决定架构**：库贵的时候大家挤一个库，库便宜的时候每个任务一个库。Supabase 赌的就是这个拐点的到来。

一句话收束：Agent 用数据库，不是因为数据库是个时髦配件，而是因为**无状态模型 + 有状态任务 + 多主体隔离**这三件事凑在一起，数据库是唯一同时满足成熟方案。

## 五、这不是一家的棋：行业格局

"每 Agent 一库"已是行业共识，分歧只在路线：

| 玩家 | 路线 | 差异点 |
|---|---|---|
| **Supabase + Turso** | Postgres + SQLite 双栈 | 入口到 PB 级全带宽；库可塞进任意沙箱，断网可读写、重连增量同步 |
| **Cloudflare** | Durable Objects + 隔离 SQLite（Agents Week 2026 推 Facets） | 算存同置、启动快（容器冷启动约 4s → 648ms），但库与 Workers 运行时绑死，文件带不走 |
| **Neon + Databricks** | Serverless Postgres | 被收购理由即"80%+ 库由 AI Agent 创建"（2025-05）；纯云端，本地运行时拿不到本地副本 |
| **AWS** | Aurora Serverless v2 / Aurora DSQL + Bedrock AgentCore | 云托管单库成本模型，走企业存量通道 |
| **PlanetScale** | 分支化 MySQL/Postgres | 偏传统托管，不主打密度；2025-11 起有 5 美元单节点 Postgres |
| **TiDB（PingCAP）** | 分布式 SQL + 向量一体化 | 面向国内 Agent 平台场景，"90% 新集群由 AI Agent 创建"（厂商口径） |

这张表暴露两个事实：其一，方向已无争议——Neon 被收购、Cloudflare 把库塞进每个 AI 生成的应用、AWS 把 serverless 讲成默认，**没人再争"Agent 要不要数据库"，争的是库的形态**；其二，形态分歧是关键变量——每个库都锁在云机房，还是每个库都能像文件一样被 Agent 随身携带。Turso 的差异化甜点正在后者。

## 六、风险与未解问题

第三方分析的担忧集中在几处，且共同点是在官方公告里全部缺位：

1. **交易条款全空白**：价格、交割日期、SLA、套餐延续承诺，公开信息一律没有；
2. **两套计费模型对不上**：Turso Developer 套餐 5.99 美元/月不限库数（2026-10-11 定价页口径），Supabase Pro 是 25 美元/月含约 10 美元算力额度、每多一个项目另计。"按项目算力"的模型撑不起千万级 Agent 库，两套价目表迟早并轨，**决定并成哪一套的权力在收购方手里**；
3. **开源中立性与引擎换代**：官方承诺"开源保持开源"，但底层引擎正从 libSQL 切到尚未 1.0 的 Rust 重写版——自托管 libSQL 的用户已经站在公司明说要离开的分支上；
4. **有前例**：2025 年 1 月 Turso 曾为新用户下线 edge replicas、移除多库 schema 与 ATTACH，当时的承诺同样是"现有负载照常"。现有负载确实被保护了，但**卖给下一个客户的产品变了**；
5. **"毕业路径"目前是句话不是工具**：SQLite → Postgres 的迁移器不存在，官宣只有方向。

务实的用户动作：导出完整库清单、在 Turso Cloud 之外留独立备份、书面索取交割日期与至少 12 个月价格锁定、合同加"控制权变更"条款。前例也有好的一面——Databricks 收购 Neon 后三个月转向按用量计费、取消月最低、扩大免费额度。但那是"有时"，不是"必然"。

## 七、判断

- **短期（对开发者）**：影响比标题小。Turso 照常跑，Supabase 不会明天换掉你的 Postgres。盯两件事即可：4.99/5.99 美元不限库的定价是否延续（延续=行业最便宜的每 Agent 一库方案），以及"毕业迁移器"是否从口号变成真工具。
- **中期（对行业）**：数据库的物理部署单位正在从"实例"变成"文件"，创建单位从"公司"变成"任务"。这把数据库竞争的赛道从"谁存储大、查询快"扩成三条：存多大、多快、**能同时养活多少个库**。
- **长期（对赌注本身）**：成立与否不取决于发布会，取决于一件朴素的事——当库的数量从百万涨到十亿，那张账单还算不算得清。Costa 的十亿库目标和 Supabase 一年 6.5 亿美元的融资，都是压在同一张账单上的注。

## 附：关键时间线

| 时间 | 事件 |
|---|---|
| 2021 | ChiselStrike 成立（Turso 前身） |
| 2022–2024 | libSQL：SQLite fork，加远程复制与向量搜索 |
| 2024-12 | 启动 Limbo：Rust 从零重写 SQLite |
| 2025-05 | Databricks 收购 Neon（"80%+ 库由 AI Agent 创建"） |
| 2026-03 | Limbo 首个支持并发写、CDC、原生向量搜索的版本发布 |
| 2026-06 | Supabase F 轮 5 亿美元，估值 105 亿美元（GIC 领投） |
| 2026-10-02 | Supabase 宣布 1.5 亿美元新融资（GIC 领投）+ 收购 Turso；同场发布 Multigres、OrioleDB |

## 参考来源

- Supabase 官方公告：*Supabase is acquiring Turso*（2026-10-02）
- Turso 官方公告：*Turso is joining Supabase to give every agent its own database*（2026-10-02，Glauber Costa）
- PR Newswire：Supabase 融资与收购新闻稿（2026-10-02）
- Supabase：*Scale without limits: Multigres, OrioleDB, and dbarena*（2026-10-02，Select 2026）
- Turso 技术博客：*Turso Cloud Goes Diskless*、*Beyond the Single-Writer Limitation*、CTO.new 客户案例、*Introducing Limbo*
- AWS Storage Blog：*How Turso built a transactional database using Amazon S3 Express One Zone*
- Databricks：*Databricks + Neon*（2025-05）
- Seedtable、Krugman Insights、datapace.ai、stackone.com、saascity.io、aibizhub.io、tech-insider.org、yage.ai、Hacker News 讨论串（2026-10）等第三方分析
