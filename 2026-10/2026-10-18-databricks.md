# Databricks 调研：Lakehouse 巨头正把自己重做为企业 AI Agent 的基础设施

> **TL;DR**：Databricks 用两年完成了从"数据湖仓公司"到"企业 AI agent 基础设施公司"的重定位——Lakebase（agent 的状态数据库）+ Genie（数据 agent）+ Agent Bricks/Omnigent（agent 构建与编排）+ Unity AI Gateway（多模型治理）构成完整栈；2026-08 以 $190B 估值再融 $5B（Coatue 领投），revenue run-rate $7B+、增速 80%+。选型一句话：企业数据与 agent 都在 Databricks 上时栈内闭环最顺，跨云多栈组织要警惕锁定与定价，纯 agent 编排需求不必买整座平台。
> **数据截至 2026-10-18。** Databricks 为未上市公司，财务数字来自官方新闻稿与媒体交叉验证，产品能力以官方文档快照为准。

---

企业客户这两年在问的问题变了。以前是"怎么把数据湖和数仓统一掉"，现在是"怎么让 AI agent 安全地读写我的业务数据、记住上下文、并且别把我的模型预算烧穿"。Databricks 的回答是整栈重做：给 agent 一个事务数据库（Lakebase）、一个理解业务数据的同事（Genie）、一个构建与编排 agent 的车间（Agent Bricks / Omnigent），以及一个管住多个模型的治理层（Unity AI Gateway）。

本文的路线：先给产品格局地图，再逐产品线同构拆解，最后横向对比、风险与未决问题、给不同读者的结论。

一句话心智模型：**Databricks 在赌"agent 时代的企业数据栈会收敛到自己身上"——数据、语义、状态、治理四层全吃，用 Lakehouse 的存量阵地做护城河。**

---

## 一、调研口径

- **信息源**：Databricks 官方新闻稿与博客（一手）、Wikipedia 公司条目（经交叉验证的二手汇总）、CNBC / Reuters / TechCrunch / SiliconANGLE 等媒体（二手）、Forbes / Fast Company 专访（二手）。
- **检索语言**：中英文。
- **免责声明**：未上市公司无审计财报，所有财务数字为公司口径 run-rate；产品 GA 边界以官方文档为准。

## 二、格局总览：三层产品地图

```
┌─ Agent 应用层 ─────────────────────────────────────────────┐
│ Genie（对话式取数）/ Genie One（agentic coworker）          │
│ Genie Code（数据工程 agent）/ Genie ZeroOps（自愈运维）      │
│ Agent Bricks（低代码构建）/ Omnigent（开源 meta-harness）    │
├─ AI 平台层 ────────────────────────────────────────────────┤
│ Mosaic AI（微调/向量检索/模型服务）                            │
│ Unity AI Gateway（多模型路由、治理、成本控制）                 │
├─ 数据底座层 ───────────────────────────────────────────────┤
│ Lakehouse（Delta Lake + SQL/ML/流，run-rate $1.5B+）         │
│ Unity Catalog（统一治理，已开源）/ Lakeflow（数据管道）        │
│ Lakebase（serverless Postgres，agent 事务状态）               │
│ Lakehouse//RT + LTAP（实时引擎 + 湖上事务分析一体）           │
└───────────────────────────────────────────────────────────┘
```

贯穿全栈的技术赌注是 **LTAP**（Lake Transactional/Analytical Processing，2026-06 发布）：把在线事务处理（OLTP）和在线分析处理（OLAP）压到同一份湖存储上，终结"为了 agent 的实时读写再维护一套独立 TP 数据库"的旧模式。这是 Lakebase 存在的架构理由，也是它对 Neon 收购（约 $1B，2025-05）的消化结果。

共同方法论一句话：**agent 需要四样东西——上下文（数据+语义）、记忆（事务状态）、工具（治理后的模型与数据访问）、护栏（成本与安全），Databricks 每层卖一件。**

## 三、逐产品线拆解

### 3.1 Lakehouse + Unity Catalog：现金牛与治理地基

- **一句话定位**：占收入大头的存量阵地，也是整个 agent 叙事的信任来源。
- **盘点**：Delta Lake（开源存储层，湖上 ACID）、Databricks SQL（数仓，2026 披露向 $1B+ run-rate 迈进）、Lakeflow（数据管道）、MLflow（开源 MLOps）；Lakehouse 产品线 run-rate 超 $1.5B、增速 100%+（2026-08 官方口径）。
- **解剖**：存算分离架构，数据留在客户自己的云存储（S3/ADLS/GCS），用开放表格式（Delta Lake / Apache Iceberg）——这是它对 Snowflake 式专有仓的核心差异，也缓解了锁定焦虑。Serverless 化是近三年重心：自动供给与伸缩，用户不再管集群。
- **安全模型**：Unity Catalog（2024-06 开源）做统一元数据与细粒度授权，覆盖表/文件/函数/模型四类资产，行级列级权限 + 审计齐备；它是上层所有 agent 工具调用的授权闸口。
- **成熟度**：完全 GA，FedRAMP 授权（2022）；是本文最不需要担心的部分。

### 3.2 Lakebase：给 agent 的记忆层，增长最快的新故事

- **一句话定位**：为 AI agent 工作负载造的 serverless Postgres，2026 年的头号战略产品。
- **盘点**：源于 Neon 收购（约 $1B，2025-05）与 Electric 收购（数据同步，2026-08）；独立 OLTP 库，与湖分析共享治理面。run-rate 已破 $1 亿（2026-08 官方口径，发布仅 14 个月）。
- **解剖**：Neon 的架构底座（分支式 serverless Postgres、按用量计费的计算分离）+ Databricks 的企业分发。对 agent 场景的针对性设计：会话状态、工具调用日志、短期记忆都可写事务库，再由 LTAP 与湖上分析打通。
- **安全模型**：继承 Unity Catalog 的授权与审计；serverless 形态下网络隔离与连接管控是部署时的主要评估点。
- **成熟度**：快速迭代中，run-rate 增速是本轮 $5B 融资的三大投向之一；Electric 的同步能力补上了"外部 Postgres → Lakebase"的迁移通道。

### 3.3 Mosaic AI + Unity AI Gateway：模型层的"瑞士"

- **一句话定位**：企业在自己数据上定制与路由模型的中间层。
- **盘点**：Mosaic AI 起于 MosaicML 收购（$1.3-1.4B，2023-06）：微调、向量检索（AI Vector Search）、模型服务与监控；Unity AI Gateway 是 2026 年的新重点——多模型接入、统一治理与成本控制。
- **解剖**：Gateway 的核心判断是"不会有单一赢家模型"：它把 OpenAI、Anthropic、Google Gemini（2025-06 四年合作协议）、开源权重模型统一在一个代理层后面，做路由、配额、预算熔断。CEO Ghodsi 公开提到客户越来越多采用中国开源权重模型——Gateway 的商品价值正来自于这种多模型现实。
- **安全模型**：模型访问统一走代理，凭据不出平台侧；PII 检测、请求审计在网关层执行；数据驻留合规随底层云账号走。
- **成熟度**：Mosaic AI GA 且已过 $1B AI 收入 run-rate（2025-09 官方口径，当时总收入 $4B+）；Gateway 处于快速产品化阶段，是融资的第二大投向。

### 3.4 Genie 家族：业务数据的自然语言入口

- **一句话定位**：让企业员工用对话拿业务数据答案的 agent，Databricks 的"门面"产品。
- **盘点**：Genie（对话取数，基于语义模型/指标层）；Genie One（2026-06 发布的 agentic coworker，扩到行动）；Genie Code（2026-03，面向数据工程与科学的 vibe-coding agent）；Genie ZeroOps（同月，数据与 AI 负载的自动监控-调查-修复）。
- **解剖**：关键在于语义层——Genie 的准确率依赖客户在 Unity Catalog 里沉淀的业务语义（指标定义、表注释、行级安全），这让"治理投入"直接变成"agent 回答质量"，是平台内生的飞轮。Genie Code 的取舍与通用 coding agent 不同：它吃透的是 SQL/Spark/管道，而不是通用代码库。
- **安全模型**：回答受 Unity Catalog 行级列级权限约束——用户无权看的数据，agent 也答不出；这是它对"把数据贴给 ChatGPT"方案的合规卖点。
- **成熟度**：Genie GA 并随平台分发；ZeroOps/Code 较新，按预览节奏迭代。

### 3.5 Agent Bricks + Omnigent：构建与编排 agent 的车间

- **一句话定位**：一个低代码构建 agent，一个开源编排 harness——两条腿走路。
- **盘点**：Agent Bricks（2025-06 发布，低代码/无代码 agent 构建器，拖出能调用平台工具的 agent）；Omnigent（2026-06 开源，定位"agent harness 的公共接口层"，在 Claude Code、Codex 等命令行 agent 之上提供统一构建、控制与协作界面）。
- **解剖**：Omnigent 是观察 Databricks agent 战略最诚实的窗口——它没有自建封闭的 coding agent，而是承认"agent 形态将由 Anthropic/OpenAI 等定义"，自己去做跨 harness 的控制平面（任务分派、审批、审计）。这与 Anthropic 买 Bun、OpenAI 买 Astral 是同一逻辑的镜像：模型厂往下买运行时，数据平台厂往上做 agent 治理面。
- **安全模型**：工具调用全部经由 Unity Catalog 授权；Omnigent 层的价值恰在于把审批与审计前置到 agent 执行前。
- **成熟度**：Agent Bricks GA 路径中；Omnigent 是新开源项目，社区 adoption 待验证。

### 3.6 Lakewatch：顺手吃掉的安全运营

- **一句话定位**：AI 驱动的安全运营平台（SOC），Lakehouse 的安全行业版。
- **盘点**：Panther Labs 收购（2026-06 宣布）+ 2026-03 发布的 Lakewatch 产品线；切入 security lakehouse 品类。
- **解剖**：安全日志天然是海量半结构化数据， Lakehouse 架构适配度高；agent 化威胁检测是差异化叙事。这是典型的"用同一底座横向打行业"打法，也是对平台粘性的加固。
- **安全模型**：继承 Panther 的检测即代码能力；作为被收购整合中的产品，部署形态与合规认证需逐项确认。
- **成熟度**：整合期，列为观察项。

## 四、横向对比

| 维度 | Databricks | Snowflake | Palantir | 云厂原生（AWS/Azure/GCP） |
|---|---|---|---|---|
| 架构哲学 | 湖仓 + 开放表格式，存算分离 | 专有数仓为主，向 Iceberg 靠拢 | 本体驱动的应用层 | 全托管服务拼图 |
| Agent 故事 | 全栈自研 + 收购补齐（Neon/Panther/Electric） | Cortex 起步较早但声量弱 | AIP 起步早，政企绑定深 | 各推各的 agent 平台 |
| 数据治理抓手 | Unity Catalog（开源） | Horizon/Polaris | 本体 + 权限映射 | IAM + 服务级策略 |
| AI 中立性 | 多模型 Gateway 是明牌战略 | 类似 | 绑定自有与 OpenAI | 绑定自家模型 |
| 云立场 | 多云中立（AWS/Azure/GCP 都跑） | 多云 | 政企私有化 | 自家云优先 |
| 成熟度 | 收入 $7B+ run-rate，80%+ 增速 | 上市公司，体量更大增速更低 | 已盈利 | 体量最大 |
| 锁定风险 | 中：格式开放、计算与元数据沉淀重 | 中高 | 高 | 中高（同云绑定） |

判断：Databricks 相对 Snowflake 的差异化在"开放格式 + AI 栈深"，相对 Palantir 在"工程自助 + 多云"，相对云厂在"中立 + 数据语义沉淀"。它的软肋同样清楚：定价随 serverless 用量模型复杂度上升，且最佳体验要求全栈押注。

**选型决策树**：数据与 AI 负载已在 Databricks → 栈内闭环，优先评估 Lakebase/Genie/Gateway；核心数据在 Snowflake 且满意 → 观望，Gateway 类能力可用独立代理层替代；强合规政企、本体需求重 → Palantir 与 Databricks 正面 PK；轻量团队无湖仓底座 → 不要为 agent 功能买整座平台，从独立工具起步。

## 五、风险与未决问题

1. **估值与 IPO 窗口**：$190B 估值（2026-08）对应约 27 倍 run-rate，一级市场溢价严重依赖 AI 叙事；S-1 未递表，IPO 时点 2026-2027 之间浮动。一旦 AI 基础设施估值回调， private mark 与公开市场的剪刀差会先于业务问题暴露。
2. **毛利率承压**：官方口径 FCF 已转正，但媒体披露毛利率在 mid-70s 且因 AI 算力成本下滑（2025 末-2026 报道）——agent 负载越是成功，推理成本越侵蚀利润，Gateway 的成本控制价值实则也是自救。
3. **锁定悖论**：开放表格式是叙事卖点，但 Unity Catalog 的语义沉淀、Genie 的调优、agent 工具链的授权模型都是迁移成本；走得越深，越离不开。
4. **与云厂的竞合**：微软合作扩到 2030s（2026-07），同时 Azure 自己有 Fabric/Copilot；多模型中立性越成功，与模型厂的利益分配越微妙（Anthropic、OpenAI 各 $100M 合作是保护费还是蜜月期，待观察）。
5. **新线产品的质量参差**：Omnigent、Genie Code、Lakewatch 都在 12 个月内发布，GA 边界模糊，把生产押在预览功能上的团队会踩坑。

## 六、观察与启示

- **收敛趋势**：企业数据平台的竞争单位从"存储/计算"上移到了"agent 工作负载"。Databricks 两年四次关键收购（MosaicML→模型、Tabular→开放表、Neon/Electric→事务库、Panther→安全）全部围绕这个迁移布点，节奏教科书级。
- **分歧本质**：agent 基础设施存在三条路线——模型厂下沉（OpenAI/Anthropic 买工具链）、数据平台上浮（本文）、云厂横扩。Databricks 的胜负手是它握着"企业语义与治理"这个另两家都不擅长的东西。
- **该借鉴**：把"治理投入"翻译成"agent 回答质量"（语义层飞轮）是把安全团队从成本中心变成 AI 项目股东的高明设计；Omnigent 的"做控制平面而非做 agent"是资源有限玩家的正确姿势。
- **该警惕**：当一家公司的产品叙事全面服务于"你已有的投入会成为 agent 的上下文"，它同时也在做"你未来的退出成本会越来越高"的复利生意。评估时把迁移演练列入年度计划，不是悲观，是对冲。

## 七、结论：给不同读者的一句话

- **开发者**：没有包袱的新数据项目用 serverless Lakehouse + Lakebase 起手，`uv`/Genie Code 这类 agent 工具先用起来再评估深度绑定。
- **架构师**：盯三个指标——Lakebase run-rate 增速（agent 状态层的真实需求）、Gateway 的多模型路由开销、以及 Unity Catalog 之外语义资产的导出成本。
- **观望者**：等两个信号——S-1 披露的毛利率走向，以及 Omnigent 社区的第三方贡献者数量（控制平面是否有标准相）。

## 附录

**A. 收购时间线**：Redash（2020，可视化）→ 8080 Labs（2021，无代码）→ Okera（2023-05，治理）→ MosaicML（2023-06，$1.3-1.4B）→ Arcion（2023-10，$100M，复制）→ Tabular（2024，>$1B，Iceberg）→ Neon（2025-05，约 $1B，serverless Postgres）→ Panther Labs（2026-06，安全）→ Electric（2026-08，数据同步）。

**B. 融资估值时间线**：$38B（2021-08）→ $43B（2023-09）→ $62B（2024-12，$10B J 轮）→ $100B+（2025-08，$1B K 轮）→ $134B（2025-12，$4B L 轮）→ $190B（2026-08，$5B，Coatue 领投；原计划募 $1B，实收认购意向约 $15B）。

**C. 关键财务节点**：FY2023 收入 $1.6B；2025-09 run-rate $4B+（其中 AI 产品线 $1B+）；2026-02 run-rate $5.4B（65%+ YoY）；2026-06 run-rate $6.9B（80% YoY，CNBC）；2026-08 run-rate $7B+。客户 20,000+；年消费超 $100 万客户 1000+；NRR >140%；FY2025 起 FCF 转正。

## 参考资料

- [Sixth Street 投资公告（2026-08-13，$5B 融资结构、参与方与三大投向，一手口径）](https://sixthstreet.com/investment_announce/databricks-grows-80-yoy-surpasses-7b-revenue-run-rate-scales-lakebase-genie-and-unity-ai-gateway/)
- [PYMNTS：Databricks 完成 $5B 融资，估值 $190B（2026-08-13，含 $62B→$134B→$190B 估值轨迹）](https://www.pymnts.com/news/investment-tracker/2026/databricks-raises-5-billion-to-expand-enterprise-ai-agent-platform/)
- [CMSWire：Databricks 营收破 $7B run-rate（2026-08，客户分层与 Lakehouse/Lakebase run-rate 明细）](https://www.cmswire.com/ai-platforms/databricks-hits-7b-run-rate/)
- [MLQ.ai：$5B 融资闭环与 AI 成本背景（2026-08-14，Ghodsi 专访要点与 Electric 收购信号）](https://mlq.ai/news/databricks-closes-5-billion-round-at-190-billion-valuation-as-ai-costs-rise/)
- [Wikipedia: Databricks 公司条目（2026-10-14 版，历史/产品/融资交叉索引）](https://en.wikipedia.org/wiki/Databricks)
- [TechCrunch：Databricks $4B @ $134B（2025-12-16，L 轮与 AI 业务线口径）](https://techcrunch.com/2025/12/16/databricks-raises-4b-at-134b-valuation-as-its-ai-business-heats-up/)
- [The New Stack：Agent Bricks 发布（2025-06-11，无代码 agent 构建器定位）](https://thenewstack.io/databricks-launches-agent-bricks-its-new-no-code-ai-agent-builder/)
- [SiliconANGLE：Databricks 与微软合作延长至 2030s（2026-07-23，Azure/Cobalt 与 Genie 进入微软产品组合）](https://siliconangle.com/2026/07/23/databricks-microsoft-expand-azure-partnership-2030s/)
