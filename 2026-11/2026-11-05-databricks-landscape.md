# Databricks 产品全景：从 Spark 公司到 AI Agent 基础设施

> 数据截至 2026-11-05。本文所有财务数字为公司官方披露的 run-rate 口径，除非另行标注；未上市公司无经审计的公开报表，数字按"公司宣称"处理。

## 一句话定位

Databricks 是以开放表格式（Delta Lake/Iceberg）为地基、以用量计费为商业模式的企业数据与 AI 平台——历史上它完成了四次定位迁移：Spark 商业化公司 → Lakehouse 开创者 → Data Intelligence Platform → AI Agent 时代的基础设施。

## 关键数字

**公司口径（官方新闻稿/财报式披露）：**

| 指标 | 数值 | 时点 |
|---|---|---|
| 估值 | 1900 亿美元（190B） | 2026-08 |
| 收入 run-rate | 70 亿+ 美元，同比 +80% | 2026-08 |
| AI 产品 run-rate | 14 亿+ 美元（占总收入约 26%） | 2026-02 |
| Databricks SQL 仓库 run-rate | 15 亿+ 美元，增速 100%+ | 2026-08 |
| 净收入留存率（NRR） | 140%+ | 2026-02 |
| 客户 | 2 万+ 组织；财富 500 强覆盖 70%；年消费 100 万美元+ 客户 1000+，1000 万美元+ 客户 100+ | 2026-08 |
| 自由现金流 | 已转正（2025 年内） | 公司披露 |

**融资与估值轨迹（第三方汇总口径）：**

- 累计股权融资约 240 亿美元（含 2024 年 100 亿美元 J 轮），含债务总融资超 320 亿美元（[multiples.vc，2026-09](https://multiples.vc/private-comps/databricks)、[TechTarget 转引，2026-08](https://tech-insider.org/databricks-valuation-175-billion-2026/)）
- 估值曲线：6.2B（2019-10）→ 28B（2021-02）→ 38B（2021-08）→ 43B（2023-09）→ 62B（2024-12）→ 100B+（2025-08）→ 134B（2025-12）→ 190B（2026-08），13 个定价轮次、估值 365 倍（[multiples.vc](https://multiples.vc/private-comps/databricks)）
- 收入曲线（融资轮同期口径）：1 亿（2019-02）→ 4.25 亿（2020 财年末 ARR）→ 6 亿（2021-08）→ 15 亿（2023-09）→ 约 30 亿（2024-12）→ 40 亿（2025-09）→ 48 亿（2025-12）→ 54 亿（2026-02）→ 70 亿+（2026-08）

一句话读表：估值 27 倍 run-rate，押的不是当期收入，是"企业 AI 负载的核心底座"这个位置。

## 产品体系：八层清单

截至 2026 年中，Databricks 的产品版图可以压缩为八层。变化速度注意点：第七、八层（Agent 与 OLTP）全部是 2025-06 之后才出现的。

**1. 开放数据底座**

- Delta Lake：开源表格式 + 事务日志（2019-04 开源，2022 年随 2.0 完全捐赠 Linux 基金会）。这是全部叙事的起点：在对象存储的廉价 Parquet 上长出 ACID、版本回滚、流批一体。
- UniForm（2023-08）：一份物理数据同时暴露 Delta/Iceberg/Hudi 三种元数据，直接回应"格式锁定"质疑。
- Unity Catalog：统一目录与治理（表、文件、模型、特征、仪表盘全对象血缘），2024-06 开源。治理是 Databricks 对企业销售的真正抓手。
- Marketplace：数据与 AI 资产交易分发层。

**2. 计算引擎**

- Photon：向量化 C++ 执行引擎（2021 GA），SQL 性能层。
- Serverless：2023 年底全面铺开的免集群模式——按秒计费、自动扩缩，是当前收入增量的主要形态。
- Lakehouse//RT（2026-06 DAIS 发布）：实时分析引擎，补齐秒级/亚秒级场景。

**3. 数据工程**

- Lakeflow（2024-10 品牌统一）：Delta Live Tables 声明式管道 + Workflows 编排。声明式"管道即代码"是卖点。
- Notebooks、Repos、Lakehouse Apps（在平台内跑第三方应用的托管框架）。

**4. BI 与对话式分析**

- Databricks SQL：数仓负载正面战场，run-rate 15 亿+ 且增速 100%+，是它从 Snowflake 嘴里抢下的一块。
- Genie：对话式取数 agent（2024 年随 AI/BI 推出），自然语言生成 SQL + 语义层校验。

**5. ML 与生成式 AI**

- MLflow（2018-06 开源）：事实标准的 MLOps 工具链。
- Mosaic AI：2023-06 以 13 亿美元收购 MosaicML 而成，覆盖模型微调、训练、推理、向量检索、Agent 评估。
- DBRX（2024-03）：开源 MoE 基础模型，造势意义大于商用。
- Unity AI Gateway：模型网关——统一接入外部大模型（OpenAI/Anthropic/Gemini 等）、做治理、限速、成本归因。

**6. Agent 平台（2025-06 起）**

- Agent Bricks（2025-06）：企业级 agent 构建/部署套件，质量评估 + 护栏 + 可观测。
- Genie 家族（2026 密集发布）：Genie One（agentic 数字同事，2026-06）、Genie Code（数据工程 agent，2026-03）、Genie ZeroOps（数据/AI 负载自动运维 agent，2026-06）。
- Omnigent（2026-06 开源）：Claude Code/Codex 这类" harness "之上的开源 meta-harness——统一接口层，把不同厂商的编码 agent 纳入同一控制平面。

**7. 事务处理（OLTP）**

- Lakebase：OLTP 数据库，底座来自 2025-05 对 Neon 的收购（报道口径约 10 亿美元），2025-06 发布预览。这是 Databricks 历史版图里缺了十年的一块：分析平台自己长不出交易系统，只能买。
- LTAP（2026-06）：Lake Transactional-Analytic Process——一份数据上同时跑 OLTP 与 OLAP 的架构声明。目前以官宣为主，工程成熟度待验证。

**8. 安全**

- Lakewatch（2026-03）：AI 驱动的安全运营平台，整合 2026-06 收购的 Panther Labs。

## 定位演进：四次迁移

Databricks 的自我定义变化是理解这家公司最好的线索。每一次迁移都不是文案更新，而是产品版图的重新划分。

**阶段一：Spark 商业化（2013–2019）**

出身 UC Berkeley AMPLab，七位创始人是 Apache Spark 的核心作者（Ghodsi、Zaharia、Stoica、Wendell、Konwinski、Xin、Tavakoli-Shiraji）。2013 年 A 轮时的公开目标是"做 Google MapReduce 的替代"（[Wikipedia 转引 2013 年报道](https://en.wikipedia.org/wiki/Databricks)）。卖的是 Spark 托管 + 企业支持。Ion Stoica 是首任 CEO，2016 年 1 月由 Ali Ghodsi 接任至今。

**阶段二：Lakehouse 开创者（2020–2023）**

2020 年 1 月官方造出 "lakehouse" 一词（后有 CIDR 2021 论文背书），2019-04 先开源 Delta Lake、2020-06 出 Delta Engine、2020-11 出 SQL Analytics——从技术栈看这是一条完整的"数据湖长出数仓能力"的进攻路线，矛头直指 Snowflake 代表的独立数仓。关键策略是**用开放格式换信任**：Iceberg 社区起来后，用 UniForm 和 2024 年收购 Tabular（Iceberg 创作者公司）把"锁定"指控就地化解。

**阶段三：Data Intelligence Platform（2023 底–2025）**

2023-12 品牌从 Lakehouse Platform 升级为 Data Intelligence Platform，同一天宣布的底气是两个月前 13 亿美元收下 MosaicML。定位话术变成"数据+生成式 AI 的统一平台"：AI/BI（2024-03）、DBRX（2024-03）、Unity Catalog 开源（2024-06）、Lakeflow（2024-10）。这一阶段开始把"企业私有数据"作为与模型厂商叫板的资产。

**阶段四：AI Agent 时代基础设施（2025–2026）**

2025-06 DAIS 是转折点：Agent Bricks、Lakebase、AI Gateway 三箭齐发；之后 12 个月内连发 Genie Code/One/ZeroOps、Omnigent、LTAP、Lakehouse//RT、Lakewatch。L 轮（2025-12）官方新闻稿的定位已经是明牌：为 AI 革命建三样东西——**AI agent 的数据库、agent 平台、数据+AI 应用**（[Databricks 新闻稿，2025-12，经 okara 汇总转引](https://okara.ai/blog/databricks-revenue-growth)）。2026 年 8 月 50 亿美元融资时，Ghodsi 对媒体的说法是：本来只想融 10 亿，投资人报了 150 亿意向，最后定了 50 亿。

## 变化时间线（2013–2026）

融资（💰）、产品（🛠）、并购（🤝）、定位（🧭）合并一表：

| 时间 | 类型 | 事件 |
|---|---|---|
| 2009–2010 | 🛠 | Spark 在 Berkeley AMPLab 诞生并开源 |
| 2013-09 | 💰 | A 轮 1390 万美元（Andreessen Horowitz 领投） |
| 2014-06 | 💰🛠 | B 轮 3300 万（NEA）；Databricks Cloud 平台发布 |
| 2016-01 | 🧭 | Ghodsi 接任 CEO（Stoica 转执行董事长） |
| 2016-12 | 💰 | C 轮 6000 万，估值 5.2 亿 |
| 2017-08 | 💰🛠 | D 轮 1.4 亿；内部启动 Delta 项目 |
| 2018-06 | 🛠 | MLflow 开源 |
| 2019-02 | 💰 | E 轮 2.5 亿，估值 28 亿（a16z） |
| 2019-04 | 🛠 | **Delta Lake 开源**——后续一切的起点 |
| 2019-10 | 💰 | F 轮 4 亿，估值 62 亿 |
| 2020-01 | 🧭 | **提出 "lakehouse" 概念** |
| 2020-06 | 🛠🤝 | Delta Engine 发布；收购 Redash（SQL 可视化） |
| 2020-11 | 🛠 | SQL Analytics 发布（后更名 Databricks SQL） |
| 2021-02 | 💰 | G 轮 10 亿，估值 280 亿（Franklin Templeton） |
| 2021-08 | 💰 | H 轮 16 亿，估值 380 亿；Photon GA |
| 2022 | 🛠 | Delta Lake 2.0 完全开源（捐赠 Linux 基金会） |
| 2023-06 | 🤝 | **13 亿美元收购 MosaicML** → Mosaic AI |
| 2023-08 | 🛠 | Delta UniForm（Iceberg/Hudi 互操作） |
| 2023-09 | 💰 | I 轮 5 亿，估值 430 亿（Capital One Ventures、NVIDIA 参投） |
| 2023-12 | 🧭 | **品牌升级为 Data Intelligence Platform**；serverless 全面铺开 |
| 2024-02 | 🤝 | 收购 Arcion（CDC 接入，1 亿美元） |
| 2024-03 | 🛠 | DBRX 开源模型；AI/BI（含 Genie 前身）发布 |
| 2024-06 | 🤝🛠 | 收购 Tabular（Iceberg 核心团队）；**Unity Catalog 开源** |
| 2024-07 | 🤝 | 收购 Lilac（数据策展）、Einblick（文本转 SQL） |
| 2024-10 | 🛠 | Lakeflow 品牌统一（DLT + Workflows） |
| 2024-12 | 💰 | **J 轮 100 亿美元**，估值 620 亿（Thrive 领投）——当时史上最大风投轮之一 |
| 2025-01 | 💰 | 52.5 亿美元信贷安排（JPMorgan 牵头部分，为 IPO 预备） |
| 2025-05 | 🤝 | **收购 Neon**（报道口径约 10 亿美元）→ Lakebase 底座 |
| 2025-06 | 🛠 | DAIS：Agent Bricks、Lakebase、AI Gateway；与 Alphabet 四年合作（Gemini 集成） |
| 2025-08 | 💰 | K 轮 10 亿，估值 1000 亿+ |
| 2025-12 | 💰 | **L 轮 40 亿+，估值 1340 亿**（Insight、Fidelity、JPM AM 领投） |
| 2026-02 | 💰 | run-rate 54 亿（+65%）；AI 产品 14 亿+；追加约 20 亿债务额度 |
| 2026-03 | 🛠 | Genie Code（数据工程 agent）、Lakewatch（AI 安全运营） |
| 2026-06 | 🛠🤝 | DAIS：Omnigent（开源）、LTAP、Lakehouse//RT、Genie One/ZeroOps；收购 Panther Labs |
| 2026-08 | 💰 | **50 亿美元融资，估值 1900 亿**（Coatue 领投）；run-rate 70 亿+（+80%） |
| 2026 年内 | 🧭 | Ghodsi：2026 是"糟糕的上市年"，IPO 最早 2027；截至 2026-08 未提交 S-1 |

## 商业模式

- **计量**：核心计费单位是 DBU（Databricks Unit），按计算类型（SQL 仓库/作业/模型训练/推理）分价，叠加 serverless 溢价。存储直接走客户自己的云账号（AWS/Azure/GCP），Databricks 不转售存储——这把"数据引力"留给了云厂，也把锁定指控减半。
- **开源漏斗**：Spark、Delta Lake、MLflow、Unity Catalog 全开源。逻辑清晰——开放格式成为行业标准后，治理层、serverless 便利性和 AI 工具链收费。Unity Catalog 开源是典型的"治理层换分发"操作。
- **增长引擎**：NRR 140%+ 说明收入主要靠存量客户扩张（用量自然增长 + 新产品交叉销售），新客获客是第二层。客户数 2 万+ 但年消费 100 万美元+ 的就有 1000+，头部集中度极高。
- **资本结构**：累计股权融资约 240 亿美元，含债超 320 亿。J 轮 100 亿里相当比例用于员工流动性（回购到期 RSU）——私有状态下留住人才的成本已经计入融资策略。2026-08 Ghodsi 自述"意向 150 亿只拿 50 亿"，与 2021 年 G 轮"目标 2 亿最后收 10 亿"同款剧本：每次都是超募，估值由投资人排队决定。
- **云厂关系**：三朵云中立分发（AWS/Azure/GCP 均可用），与 Microsoft 绑定最深（2017 年即有 Azure Databricks）。合作拿分销，竞争在 serverless 算力层——Databricks serverless 跑在自己的云账号里，绕开云厂数仓定价。

## 竞争坐标与非目标场景

**正面战场：**

| 对手 | 战线 | 态势（2026 年视角） |
|---|---|---|
| Snowflake | 数仓 + AI | 互相侵入对方腹地。Databricks SQL run-rate 15 亿+、增速 100%+；Snowflake 产品收入增速 26%（2026 财季口径）。Databricks 估值约 2 倍于 Snowflake 市值，增速差是主因 |
| Palantir | 决策层/本体 | Palantir 卖决策层，Databricks 卖数据+AI 底座，项目制 vs 平台制 |
| 微软 Fabric | 云厂全家桶 | 生态绑定型竞争；Databricks 的优势是跨云中立 |
| 云厂 AI 栈（SageMaker/BigQuery/Vertex） | AI 平台 | Databricks 的客户资产是"私有数据在哪"，不是模型 |
| Starburst/Dremio/Trino 生态 | 开放湖查询 | 被 UniForm + Unity Catalog 开源对冲中 |

**不是它目标的场景：**

- 中小团队快速起后端（Supabase/Firebase 价位带）：Databricks 没有免费档，入场就是企业预算。
- 轻量边缘/嵌入式 OLTP：Lakebase 走云端 serverless Postgres 路线，不碰嵌入式与局域网。
- 纯 LLM 应用原型：那是一众 BaaS 和 Vercel 生态的价位带，Databricks 的入场券是"已有企业数据"。

**风险提示**：估值 27 倍 run-rate 依赖增长不减速；Lakebase/LTAP 的 OLTP 能力来自收购，与原生 OLTP 厂商的工程积累存在代差风险；2027 IPO 窗口若遇市场转向，超募模式会反噬。

## 判断

1. **护城河在治理层和分发，不在 Spark。** 分析引擎 Photon 很强但可替代；Unity Catalog 成了企业数据资产的事实目录层、Delta 成了默认表格式——这两个"默认"才是定价权的来源。
2. **两年完成版图再造，但 OLTP 是买来的时间。** Lakebase/LTAP 补上了分析平台最难自给的一块，代价是 Neon 的团队整合。买来的 OLTP 能否扛住企业核心交易负载，是未来 24 个月最值得盯的验证点。
3. **IPO 是最大变量。** 2026 年超募 50 亿等于把上市时钟拨到 2027。若届时增长仍在 50%+ 且 FCF 为正，这将是大模型时代企业软件最大 IPO；若 AI 收入占比提升但估值倍数先坍缩，则 27 倍 run-rate 会是招股书里最难看的一行。

## 参考

- [Databricks funding history（multiples.vc，2026-09）](https://multiples.vc/private-comps/databricks)
- [Databricks 公司史与融资表（Wikipedia，2026-10 更新）](https://en.wikipedia.org/wiki/Databricks)
- [Databricks 估值与收入分析（okara.ai，2026-10-19）](https://okara.ai/blog/databricks-revenue-growth)
- [Databricks $190B 估值与 IPO 时钟（tech-insider.org，2026-10-05）](https://tech-insider.org/databricks-valuation-175-billion-2026/)
- [Databricks IPO 与 Series L 细节（tech-insider.org，2026-09-11）](https://tech-insider.org/databricks-134-billion-ipo-enterprise-software-2026/)
- [Lakehouse 概念起源（TechTarget，2020-12-29）](https://www.techtarget.com/searchdatamanagement/feature/Emerging-data-management-trends-to-watch)
- [Databricks 简史（bigeye，2023-09-08）](https://www.bigeye.com/blog/a-brief-history-of-databricks)
- [Delta Lake 论文与技术脉络（datasystem.school）](https://datasystem.school/data-system-papers-distilled/papers/delta-lake.html)
- [Databricks 融资简表（getlatka，2026-10-07）](https://getlatka.com/companies/databricks/vs/ssi.inc)
