# TiDB AI 功能全景：存量能力盘点、2026 年新增清单与未来趋势判断

> 一句话结论：TiDB 的 AI 故事不是"加一个向量索引"，而是把向量检索、全文检索、HTAP 分析和事务放进同一个分布式引擎，再以 TiDB X 新架构把云的弹性做到"缩容至零"，目标很直白——做 AI Agent 的数据底座。2026 年的增量几乎全在云侧（TiDB Cloud Lake、PG 兼容、BYOC、平凯 DBaaS），自管理内核全年只有 8.5.x 维护性更新。

**TL;DR**

- AI 功能分六块：向量检索、全文/混合检索、Chat2Query（NL2SQL）、TiDB MCP Server、pytidb SDK、面向 Agent 负载的 Cloud 形态
- 2026 年新增：TiDB Cloud Lake（6-30 公测）、PG 兼容 Starter（pgvector 兼容，年内公测）、全文检索上云扩区（5-6 月）、BYOC on AWS（10 月，TiDB X 内核）、国内平凯 DBaaS（4-8）、MCP Server 开源（4 月文档）
- 自管理内核：2026 全年停在 v8.5 LTS 线（8.5.5→8.5.8），无 9.x，向量检索在自管理侧文档仍标"公测"
- 未来主线判断：TiDB X（对象存储底座 + 存算分离）承接 AI 负载；多协议（MySQL + PG）抢生态；定位从"HTAP 数据库"改写为"Agent 记忆体与数据基础设施"

---

## 一、AI 功能全景：六块能力

### 1. 向量检索（Vector Search）

TiDB 的向量能力是内核级集成，不是外挂插件：

- **数据类型**：`VECTOR` 类型，单向量最大 16383 维
- **索引**：HNSW 近似最近邻索引，构建在 TiFlash 列存副本上；建表时声明向量索引，TiDB 自动为该表 provision TiFlash 副本
- **距离函数**：`VEC_COSINE_DISTANCE()`、`VEC_L2_DISTANCE()`
- **事务一致性**：向量与其描述的业务行在同一个引擎内提交，无双写问题——这是"关系库内生长量检索"相对独立向量库的结构性差异
- **版本要求**：自管理需 v8.4.0+（推荐 v8.5.0+）；TiDB Cloud Starter 直接可用
- **Auto Embedding**：支持纯文本直写直查，由后台 pipeline 自动完成文本到向量的转换，应用侧无需自己接 embedding 模型
- **现状标注**：官方文档仍将向量检索标注为**公测（preview）阶段**（数据截至 2026-10），自管理侧尤其谨慎

### 2. 全文检索与混合检索（Full-Text / Hybrid Search）

- **MySQL 兼容侧**：全文检索以公测形态存在于 8.x 内核
- **PG 兼容侧（2026）**：PG 兼容的 TiDB Cloud Starter 支持 `tsvector`/`tsquery`、语言分词器、排名函数、GIN 索引；内置中文分词（`jieba`/`zhparser` 兼容配置、`chinese_ngram`），不要求单独装扩展
- **混合检索**：向量（语义）+ 全文（关键词）+ 标量过滤可在同一 SQL 内表达，这是 RAG 场景提升召回质量的标准做法，TiDB 将其收敛为单库方案，不用 ES + 向量库 + OLTP 三件套

### 3. Chat2Query（NL2SQL）

- 2023-01 上线，基于 GPT 系列模型，用自然语言生成 SQL 并在库内执行
- 官方披露的基准成绩：BIRD 榜单 58.15 / 60.98（2024 年口径）、Spider 86.30（2025 年口径）——**厂商口径，未经独立复现**
- 提供 Chat2Query API：自动生成 Chat2Query 系统的 Data App 与 Chat2Data endpoint
- 现状：产品页仍标 preview，2026 年未见 GA 或大版本更新，演进优先级明显让位于向量检索与 Agent 基础设施

### 4. TiDB MCP Server

- 开源项目，官方文档更新于 2026-04-24，把 TiDB 接入 MCP（Model Context Protocol）生态
- 暴露的工具：`show_databases`、`switch_database`、`show_tables`、`db_query`、`db_execute`、`db_create_user`、`db_remove_user`
- 即：Agent 框架可通过 MCP 直接对 TiDB 做 schema 发现、只读/写查询和用户管理，是"数据库作为 Agent 工具"的标准化接法

### 5. pytidb：官方 Python AI SDK

- 定位"TiDB AI SDK"：连接 TiDB、定义表结构即自动建表
- 支持 Vector / Full-text / Hybrid / Image 四类检索，内置自动 embedding
- 官方示例覆盖 AI Agent Memory（跨会话持久记忆）、RAG、以图搜图等；GitHub 仓库 `pingcap/pytidb`

### 6. 面向 Agent 负载的 Cloud 形态

- TiDB Cloud Starter（Serverless）定位"Agent-ready"：免费起步、秒级创建、按 RU 计费
- GitHub 主仓库的品牌语已改为 "built for **agentic workloads** that grow unpredictably"——配合 ACID、事务/分析/向量原生支持做卖点
- 头部 Agent 产品采用案例：2026 年公开报道中，Kimi K2.6 Agent 建站服务等将 TiDB 作为数据底座（厂商侧传播口径）

---

## 二、2026 年新增清单（时间线）

| 时间 | 事项 | 性质 |
|---|---|---|
| 2026-01-15 | 自管理 v8.5.5 发布（8.5 LTS 维护线） | 维护更新 |
| 2026-04-08 | 平凯数据库（国内商业品牌）推出全托管 DBaaS：与 TiDB Cloud 同源内核，存算分离、秒级扩缩容、"缩容至零"，宣称 TCO 降 50%；**新内核一体化融合 HTAP、向量存储与全文索引** | 云产品，AI 底座定位 |
| 2026-04-14 | TiDB Cloud 控制台统一改版；**TiDB X 项目支持**（TiDB X = 基于 TiDB X 架构的服务化形态，Starter/Essential 即 TiDB X 实例）；默认版本升 v8.5.6 | 架构落地 |
| 2026-04-24 | TiDB MCP Server 文档/开源更新 | Agent 生态接入 |
| 2026-05-26 | TiDB Cloud Starter 全文检索（公测）扩区：新增东京、俄勒冈 | AI 检索能力上云 |
| 2026-06-09 | 全文检索再扩至弗吉尼亚，累计 5 个 AWS 区域 | 同上 |
| 2026-06-30 | **TiDB Cloud Lake 公测**：云原生分析数仓，面向"现代分析与 AI 数据工作流"，单服务内含弹性数仓、ANSI SQL、对象存储、全文检索、向量检索、地理空间分析 | 年度最重要的 AI 相关产品 |
| 2026-06 | Top RU 公测（分钟级 RU 消耗 Top SQL，成本可观测） | 配套 |
| 2026-08-18 | External Function / UDF 体系（可对接外部服务，含 AI 服务调用的想象空间） | 平台能力 |
| 2026-08-27 | 自管理 v8.5.8（8.5 LTS 最新补丁） | 维护更新 |
| 2026-09-17 | 自管理 v7.5.8（旧 LTS 线收尾） | 维护更新 |
| 2026 年内 | **PG 兼容 TiDB Cloud Starter 公测**（AWS）：创建实例可选 MySQL 或 PG 兼容，支持 pgvector 兼容的向量检索与部分 PG 扩展 | 多协议战略 |
| 2026-10 | **TiDB Cloud BYOC on AWS 公测**：由 TiDB X 内核驱动，明确面向关键任务负载与 **AI 负载** | 企业级交付形态 |

同期背景事件（非功能，但影响判断）：2026-10 发生 TiDB Cloud 安全事件，企业客户信任重建是 PingCAP 当前的现实约束。

**自管理内核的空白值得单独说**：截至 2026-10，自管理版本最新仍是 8.5.8（2024-12 发布的 8.5.0 LTS 的补丁线），2026 全年没有 9.x DMR，也没有新 LTS。AI 相关的内核创新（TiDB X、一体化向量+全文新内核）优先落在云形态，自管理侧向量检索文档仍挂"公测"。这是观察 PingCAP 资源分配的最直接信号。

---

## 三、未来趋势判断

以下分"事实延伸"与"推断"两档，推断处单独标注。

### 1. TiDB X 成为云侧绝对主线（事实延伸）

TiDB X 架构（2025-10 首次官宣）用对象存储重做存储底座，实现彻底存算分离。2026 年内它已承接 Starter、Essential、BYOC，并被官方描述为"面向关键任务和 AI 负载"。"缩容至零 + 秒级扩容"对 Agent 工作负载（流量不可预测、空闲期长）是直击痛点的卖点。预计后续 Dedicated 线逐步并入 TiDB X，老架构退场。

### 2. 多协议：MySQL 之外吃下 PG 生态（事实延伸 + 推断）

PG 兼容 Starter 公测、pgvector 兼容、PG 扩展支持——这是 2026 年战略级的一步。PG 是 AI 应用生态的事实中心（LangChain/LlamaIndex 默认示例、Supabase/Neon 等），TiDB 用"PG 协议 + 分布式扩展 + HTAP"切这个池子。**推断**：2027 年 PG 兼容走向 GA，并下沉到自管理内核。

### 3. 定位话术全面转向 Agent 基础设施（事实延伸）

2026-06 PingCAP 在公开技术分享中明确讲"记忆层新框架"：Agent 的长短期记忆存取、超长上下文管理、多 Agent 协作一致性、非结构化知识实时检索，全部收敛到一个 All-in-One 数据库。配套动作是 MCP Server、pytidb 的 Agent Memory 示例、"agentic workloads" 品牌语。这是把"HTAP"叙事平移成"Agent 数据底座"叙事，对标的是 Mem0/Zep 这类记忆层 + 向量库的组合方案，卖"一个库替代一组组件"。

### 4. 自管理内核 AI 能力 GA 节奏偏慢（事实 + 风险）

向量检索自管理侧公测已超一年（2024-11 随 8.4.0 DMR 引入），到 2026-10 仍未 GA，且无 9.x 时间表。**推断**：PingCAP 商业化重心在云上，开源内核的 AI 功能更多承担"能力证明"角色。对自部署用户，这是现实的时间成本风险。

### 5. 竞争挤压方向（推断）

- 上探：PostgreSQL + pgvector/pgvectorscale/VectorChord 生态迭代极快，托管 Neon/Supabase 抢 AI 应用默认选型
- 侧压：Apache Doris 4.0（2025-10 GA）已把向量+全文+BM25+AI 函数做进分析引擎，OB、GoldenDB 在国产替代场景贴身竞争
- TiDB 的差异化仍在"事务+分析+向量同引擎、分布式水平扩展、MySQL/PG 双协议"，但这个窗口期不超过 12-18 个月

### 6. 安全与合规成为企业交付前提（事实延伸）

10 月安全事件之后，BYOC、专属部署、国产合规版本（平凯）的重要性上升。AI 功能再怎么讲，企业采购的第一问会变成数据边界。

---

## 四、速查：一个 AI 应用选 TiDB 的决策点

| 场景 | 建议形态 | 说明 |
|---|---|---|
| 原型 / 小流量 Agent 应用 | TiDB Cloud Starter（免费层） | 秒级创建，向量+全文可用，RU 计费 |
| PG 生态工具链 | PG 兼容 Starter（公测，勿上生产） | pgvector 兼容，年内公测 |
| 分析 + 检索一体 | TiDB Cloud Lake（公测） | 数仓/全文/向量/地理单服务 |
| 企业关键负载 | BYOC on AWS（公测）或国内平凯 DBaaS | TiDB X 内核，自主云账号 |
| 自管理部署 | v8.5.x + 向量公测功能 | 接受 preview 状态，无 9.x 时间表 |
| Agent 框架接入 | TiDB MCP Server + pytidb | 标准化工具协议 + Python SDK |

---

## 附：数据来源与口径说明

- 官方文档（docs.pingcap.com）：向量检索、全文检索、Chat2Query、MCP Server、TiDB Cloud Release Notes（2026）、自管理版本列表
- GitHub：pingcap/tidb、pingcap/pytidb
- 第三方：2026 中国数据库流行度月报（modb）、A2M 技术分享预告（2026-06）、OSCHINA 案例报道（2026-05）
- 所有基准分数（BIRD/Spider）与 TCO 数字（-50%）均为厂商口径
- 数据截至：2026-10-26
