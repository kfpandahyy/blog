# 本体与知识图谱调研：RAG 答不了"谁依赖谁"，答它靠图

> 类型：技术点调研（按 `templates/tech-topic-research.md` v3）
> 数据截至 2026-10，GraphRAG 生态迭代极快，版本号以 GitHub 为准

**TL;DR**
- 本体是某领域概念体系的显式规范（概念、关系、约束），知识图谱是按规范填充的实体关系实例数据——前者是图纸，后者是楼。
- 2024 年 GraphRAG 把知识图谱带回 AI 主场，2025 年证据显示它并非 RAG 的升级版：简单查询上可能反输朴素 RAG。2026 年的共识是"按需用图"——多跳关系、可解释推理、决策校验才上图。
- 该做什么：从轻量属性图起步（LightRAG 一天可跑通），把本体当治理资产而非搜索功能；查询类型决定结构，不要为上图而上图。

---

开篇。你的 RAG 系统能答"哪份文档提到过供应商风险"，但答不好"这个变更会影响哪些客户、经由哪些依赖"——后者不是检索问题，是遍历问题。知识图谱就是为遍历而生的结构；本体是它背后那套"什么是什么、什么和什么该怎么连"的正式约定。这两个词出自语义网时代，曾被视为过度工程化的学术遗产；2024 年微软 GraphRAG 发布后它们被重新请回 AI 主场，一年之内又经历了退烧与重新定位。本文按"定义→何时用→演进→原理→模式→生态→对比→实践→坑"的顺序拆解，最后给三条可执行结论。

一个贯穿类比：**向量库像按颜色找照片，知识图谱像地图。前者答"哪张最像"，后者答"从这里怎么走、中间经过谁"。本体就是地图的图例和制图规范**——没有它，地图只是连了线的点。

## 一、定义与边界

一句话定义：**本体（Ontology）是对某领域概念体系的显式规范——类、关系、属性、公理与约束的正式定义；知识图谱（KG）是按这套规范组织起来的实例数据，以"实体-关系-实体"三元组（或属性图的节点-边）构成可查询、可推理的关系网络**（Gruber 1993：本体是"概念化的显式规范"）。

最小真实实例。属性图里的一条事实：

```cypher
(:公司 {name:"华为"})-[:发布]->(:产品 {name:"Mate 80"})
```

"华为发布了哪些产品、各自的供应商是谁"是一次二跳遍历——SQL 里要两次 JOIN 还要预先知道表结构，图查询直接沿边走。

六个相邻概念辨析（定义不清的争论大多源于混用）：

| 概念 | 是什么 | 关键判据 |
|---|---|---|
| 分类法 Taxonomy | 树形分类体系 | 只有 is-a 层级，无关系与约束 |
| 本体 Ontology | 带公理与规则的领域词汇表 | 机器可校验、可推理："这两条边不该同时存在" |
| 知识图谱 KG | 按模式填充的实例数据 | 能答"具体哪个实体、什么关系" |
| 语义层 Semantic Layer | 业务指标到物理数据的翻译层 | 管"GMV 口径"，不管实体间关系 |
| 向量库 Vector DB | 嵌入相似度索引 | 答"像什么"，不答"是谁、连向谁" |
| RAG | 检索增强生成范式 | 把找到的文本片段塞进上下文 |

Neo4j 官方博客（2026-09）有一个简洁的归纳：**KG 是你建造的东西，taxonomy 和 ontology 是图纸**；很多 KG 从轻模式起步，只在需要形式语义或跨系统互操作时才引入完整本体。实践中还有一个 useful 的层次栈：taxonomy（分类）→ ontology（关系+约束）→ KG（实例数据）→ semantic layer（指标口径）→ context graph（为 Agent 装配的上下文图，见[第 5 节](#五使用模式拆解)）。

## 二、何时用、何时不用

**适用：**

1. **关系与多跳是核心**：影响面分析（改这个字段波及哪些下游）、供应链追溯、欺诈环检测、"我同事的同事的供应商"。
2. **需要可解释推理**：答案是路径不是片段，每条边有语义有出处，监管和审计场景必备。
3. **实体对齐与主数据**：同一客户在公司五个系统里五个名字，图谱做实体消解，RAG 做不到。
4. **跨系统语义一致性**：多团队多模型消费同一套业务概念，本体是唯一可信词汇表（你的 AWS 调研里 `business_concepts` 那层，本质就是轻量本体）。
5. **AI 决策需要约束与权限**：本体不仅是知识，还定义"哪些动作合法"（Palantir 路线，见 5.2）。

**不适用（诚实陈述）：**

1. **查询以语义相似为主**（"帮我找类似的案例"）——向量检索更简单更便宜，上图是负资产。
2. **数据量小、关系浅**——三跳以内 SQL JOIN 足够，图的运维复杂度不划算。
3. **没有领域专家维护模式**——本体工程的成本大头是"人和共识"，不是软件。
4. **追求开箱即用的轻量问答**——GraphRAG 索引期要烧 LLM token，轻量场景 vanilla RAG 反而更好（有 2025 年实证，见[第九节](#九失败模式与坑)）。

**核心 trade-off**：图谱用建模成本换查询精度与可解释性。本体越形式化（OWL 公理、SHACL 约束），机器可校验性越强，但建模与维护成本越高。LLM 自动抽取大幅降低了"建"的成本，却把噪声带进了"准"——这是 2026 年这个领域最现实的一对张力。

## 三、演进脉络

```text
1993  Gruber 提出本体定义（"概念化的显式规范"）
2001  W3C 语义网：RDF/OWL/SPARQL 标准族，Tim Berners-Lee 推动
2007  Neo4j 创立；2011 Cypher 语言；属性图路线与 RDF 路线分庭抗礼
2012  Google Knowledge Graph 上线（"things, not strings"）；Wikidata 成立
2012- 企业 KG 黄金期：Palantir Gotham/Foundry 本体、金融 FIBO、医疗 SNOMED/FHIR
2023  SQL/PGQ 并入 ISO 9075；图查询开始回归 SQL 体系
2024  GQL 成为 ISO/IEC 39075:2024 标准（4 月发布，Neo4j 与 AWS 联合推动）
2024  Microsoft GraphRAG（arXiv 2404.16130，2 月预印/7 月开源）——LLM×KG 引爆点
2024-25 轻量派爆发：HippoRAG（2024-05）、LightRAG（2024-10）、蚂蚁 KAG/OpenSPG（2024-10）
2025  云厂商托管化：Bedrock GraphRAG GA（2025-03）、Snowflake Cortex GA（2025-11）
2025  退烧证据：GraphRAG 在简单查询上反输 vanilla RAG（Han et al. 2025 等）
2026  分化定型："按需用图"（Use Graph When It Needs，2026-02）；企业侧本体
      作为"决策层"走红（Palantir AIP 模式）；GQL 生态落地
```

**路径选择**：语义网路线（RDF+OWL，重形式化、强互操作、学术与政企合规场景）与属性图路线（轻模式、工程友好、互联网与 AI 场景）分化了二十年，2026 年的现实是——AI 应用几乎一边倒走属性图，RDF 阵营守住金融合规和生命科学。新人从属性图入门，两条路不冲突（RDF 可转属性图，neo4j-neosemantics 等工具成熟）。

## 四、解剖：架构与原理

本节用一个贯穿实例：给一组企业文档（合同、工单、组织架构）建一个"客户-产品-依赖"图谱，支撑"变更影响谁"的查询。

### 4.1 数据模型：三元组与属性图

```text
RDF：  <华为> <发布> <Mate80> .              # 三元组，谓词也是 URI，语义最严
属性图：(华为:公司 {industry:"ICT"})-[发布 {date:"2026-09"}]->(Mate80:产品)
       # 节点和边都可以挂属性，工程上更直观
```

RDF 的优势是每个谓词全局唯一（URI），跨系统拼接无歧义——代价是表达繁琐、学习陡。属性图牺牲全局语义换取表达力与性能，Cypher 的 ASCII-art 语法（`(a)-[:KNOWS]->(b)`）把"白板上的图画"直接变成查询。**设计决策一：为什么需要形式化本体而不是把约束写进 prompt？** 因为软约束会被概率生成绕过：Neo4j 团队实测"70% 的可修复错误源于约束定义不清"，SHACL 之类的硬约束在写入时机械拒绝，与模型输出质量解耦——这是"本体校验"对"提示词祈祷"的本质优势。

### 4.2 本体工程：类、关系、公理

一个最小本体要回答四个问题：有哪些类（客户/产品/服务）、类之间允许哪些关系（客户-订阅-服务）、属性取值约束（服务等级 ∈ {金牌, 银牌}）、公理（订阅是双向的吗？是否传递？）。行业本体提供了跨组织复用的答案：金融 FIBO、医疗 SNOMED CT 与 HL7 FHIR、互联网 Schema.org。推理机（RDF 阵营的 OWL reasoner）能算出"演绎闭包"——显式存 30% 的边，推出其余 70%；属性图阵营通常不做全局推理，靠查询时遍历（工程上的务实妥协：大部分 AI 应用要的是"查得到"，不是"推得出"）。

### 4.3 构建流水线：LLM 如何改变建图成本

传统 KG 构建是 NLP 流水线（NER → 关系抽取 → 实体消解），人工标注重、领域迁移差。2024 年后的范式是 LLM 抽取：

```text
文档 → chunk → LLM 按预定义 schema 抽取 (实体, 关系, 实体, 出处) →
实体消解（这个"华为"和那个"华为"是不是同一个）→ 写图（带置信度与 provenance）
```

关键改进是"按预定义 schema 抽取"而不是开放式抽取：Anthropic 的伊拉克马穆纳研究项目发现，提供领域本体后再让模型抽取，质量与一致性显著提升——**本体定义查询什么，图谱才能答什么；没有预定义模型，LLM 会抽取出无法对齐的无谓变化**。蚂蚁 KAG（2024-10）把这一步推到极致：用概念语义约束（ CN 概念-属性图双模态）引导抽取，解决 LLM 抽"类似但不等价"关系的过度泛化问题。

### 4.4 GraphRAG 索引与查询：成本花在哪

微软 GraphRAG（2024）的索引期是全文档 LLM 抽取 + Leiden 社区发现 + 逐社区递归摘要；查询分三种模式：local search（社区+chunk）、global search（全社区摘要 map-reduce，答"全体主题"类问题）、DRIFT search（先全局定位再局部展开）。**设计决策二：为什么 GraphRAG 这么贵？** 索引期对每份文档做全量 LLM 抽取和多轮社区摘要，官方文档给出的量级是每百万 token 约 4-7 美元；对 100 份 32k 词文档，全量索引几百美元、几十分钟到几小时，且每次数据更新都要重跑（早期版本）。LightRAG（HKUDS，arXiv 2410.05779）把成本压到每文档约 0.15 美元：只抽实体关系，不做社区摘要，检索时双层取（low-level chunk + high-level 图邻域），检索 token 比 GraphRAG 低约三个数量级，且天然增量更新。HippoRAG（OSU）走另一条路：检索时用 Personalized PageRank 在图上做多跳扩散，把"组合多个事实回答多跳问题"变成图算法问题，PPR 一跳展开就能串起完整推理链，且零 LLM 调用。

### 4.5 KG × LLM 的四个结合点

```text
① LLM 建 KG：抽取、消解、对齐（成本↓ 噪声↑）
② KG 增强检索：GraphRAG 家族（多跳、全局主题、可解释）
③ KG 约束生成：本体/SHACL 校验 LLM 输出，Palantir 决策校验属此类
④ KG 作 Agent 记忆：Graphiti（Zep）时序图谱，持续吸收对话与数据变更
```

**设计决策三：为什么 2026 年行业讲"按需用图"而不是"全面 GraphRAG"？** 因为有反面证据：Han et al.（2025）等在真实任务上测得 GraphRAG 在简单事实查询上显著掉点、延迟大幅增加，硬套图反而不如朴素 RAG；2026 年 2 月的后续研究（Use Graph When It Needs）提出先分类查询（事实型/多跳型/全局型），只对多跳与全局型启用图检索。**图是特定查询类型的工具，不是 RAG 的替代品**——这是退烧后沉淀下来的正确认识。

## 五、使用模式拆解

### 5.1 检索增强模式（GraphRAG 家族）

- **是什么**：文档进图谱，查询时检索子图而非文本块，支持多跳与全局摘要。
- **何时用**：多跳关系问题、跨文档全局主题（"这些工单暴露了哪些系统性风险"）、需要答案附带关系路径。
- **示例**：`pip install lightrag-hku`，几十行接入 OpenAI 兼容端点即可对文档集建图检索（官方 README 有一分钟上手的 API 调用）。
- **注意**：索引成本与更新频率先算账；简单问答占比高的语料不要全量上图。

### 5.2 决策与操作模式（Palantir 本体）

- **是什么**：本体不只是知识，还是操作系统的对象模型——对象类型（名词）+ 动作类型（动词）+ 函数 + 权限，AI 在受约束的动作空间内决策。
- **何时用**：企业要把 AI 从"问答"推向"执行业务动作"，且必须可审计、可回滚、权限内闭环。
- **示例**：Palantir AIP 的模式：Ontology 定义合法动作集 → LLM 生成候选决策 → 平台对照本体约束校验 → 通过才执行，全程留痕；Ontology MCP Server 把它暴露给外部 Agent（你的 AWS 调研里 `business_concepts` 同源）。第三方拆解（2026）指出其实施周期约 6-12 周，且数据格式开放但治理深度绑定平台。
- **注意**：这是平台级投入，不是功能级投入；"本体表示决策而不仅是数据"是它与普通 KG 的本质区别。

### 5.3 记忆模式（Agent 结构化记忆）

- **是什么**：把 Agent 的交互历史持续抽成时序知识图谱，实体关系随时间演化、可增量化更新。
- **何时用**：Agent 需要长期记住"谁、什么时候、说了什么、后来变了"，数据高频更新（传统批量 GraphRAG 做不到增量）。
- **示例**：Zep 的 Graphiti（arXiv 2501.13956，Neo4j 官方博客 2025-03）：混合检索（向量+BM25+图遍历），检索 P95 约 300ms，检索过程零 LLM 调用；Mem0 的 Mem0g（arXiv 2504.19413）用图替代平面记忆列表提升关系推理。
- **注意**：与"上下文与记忆工程"直接相关——KG、向量库、Skill 是长期记忆三种实现形态中的结构化一支（参看你知识库里 Agent Harness 与 Agent Skill 两篇）。

### 5.4 平台底座模式（生产级 KG 栈）

- **是什么**：把图谱当企业数据基础设施，服务主数据、数据治理、语义搜索多租户复用。
- **何时用**：受监管行业（金融、医药）、多系统实体对齐、跨团队语义一致性。
- **示例**：DataPraxis（2026-06）梳理的生产栈七层很有参考价值：①RDF 存储（Stardog、GraphDB、Neptune）②属性图存储（Neo4j、TigerGraph、Memgraph）③混合多模（Neptune Analytics：RDF+openCypher+向量 HNSW）④虚拟化联邦（Ontop、Denodo）⑤实体消解（Senzing、Tamr、Zingg）⑥LLM 抽取（GraphRAG、LightRAG、Graphiti）⑦治理元数据（DataHub、Atlan、Collibra）。一个受监管的中型银行生产 KG 通常触达其中四层。
- **注意**：绝大多数需求只用到其中两三层，别按七层全建。

## 六、生态格局

| 层 | 代表 | 锚点 |
|---|---|---|
| 标准 | W3C RDF 1.1/OWL/SHACL/SPARQL；ISO/IEC 39075:2024 GQL（2024-04，Neo4j+AWS 联合）；SQL/PGQ（ISO 9075:2023） | GQL 是图查询语言的 SQL 时刻 |
| 开源框架 | MS GraphRAG（2024-07 开源）、LightRAG/HKUDS（2024-10，EMNLP 2025）、蚂蚁 KAG+OpenSPG（SPG 2023-08 开源，KAG 2024-10 产品化）、HippoRAG（2024-05）、Graphiti/Zep（2025-01 论文） | LightRAG 以结构简洁+增量友好取胜；KAG 走中文专业领域的 schema 中心路线（GitHub 7.6k stars） |
| 图数据库 | Neo4j、Amazon Neptune、TigerGraph、Memgraph、ArangoDB；RDF 系 Stardog、GraphDB、AllegroGraph；国产：NebulaGraph、HugeGraph、Ultipa、Galaxybase | Neo4j 生态最成熟；Neptune Analytics 一条产品线同时吃 RDF/属性图/向量 |
| 云托管 GraphRAG | Bedrock GraphRAG（GA 2025-03）、Snowflake Cortex（GA 2025-11）、Vertex AI + Spanner Graph | 托管化说明图检索已进入企业采购清单 |
| 企业本体平台 | Palantir Foundry/AIP、Stardog、Atlan/Collibra（治理） | Palantir 是"本体=决策层"路线的标杆 |
| 公共 KG | Wikidata（2012）、ConceptNet、WordNet、DBpedia | 通用领域起点与对齐锚 |

## 七、对比评估

**RDF 路线 vs 属性图路线**：

| 维度 | RDF/OWL | 属性图 |
|---|---|---|
| 语义严格度 | 全局 URI，跨系统无歧义 | 局部标签，靠约定 |
| 推理 | OWL 推理机、演绎闭包 | 一般查询时遍历 |
| 学习成本 | 高（SPARQL+描述逻辑） | 低（Cypher 类 SQL） |
| 典型场景 | 合规、生命科学、跨机构 | AI 应用、主数据、反欺诈 |
| AI 生态 | 薄 | 厚（GraphRAG 家族几乎全在属性图上） |

**KG 检索 vs 向量检索**：

| 维度 | 向量/RAG | 知识图谱 |
|---|---|---|
| 回答 | "哪段文本像答案" | "哪些实体如何相连" |
| 多跳 | 弱（跨片段组合靠 LLM 运气） | 强（图遍历一跳一跳走） |
| 可解释 | 弱（相似度分数） | 强（路径即证据链） |
| 更新 | chunk 级增删容易 | 实体级增量可做但工程复杂 |
| 成本 | 低 | 索引期高，查询期可很低 |

**决策建议**：查询以语义相似为主 → 向量检索；出现"影响哪些/经由哪些/谁和谁什么关系"且答错代价高 → KG；两者都有 → 混合检索（向量找入口，图做多跳），这正是 Graphiti、Neptune Analytics 和各家 2026 年产品的默认答案。买商用平台前先问：是否支持增量更新？实体消解怎么做？边的 provenance 存不存？

## 八、实践：上手与最佳实践

**最小路径**（本地一天跑通 LightRAG，真实命令）：

```bash
pip install lightrag-hku
# OpenAI 兼容端点 + 一个 working_dir，即可对文档集建图检索；
# 检索自动同时查向量 chunk 与图邻域，增量插入新文档无需全量重建。
```

更进一步：Neo4j 提供 LLM Knowledge Graph Builder（网页端拖入文档自动建图）；企业级评估蚂蚁 OpenSPG/KAG（中文专业领域、schema 中心，医疗/政务/金融已有落地案例）。

**评估方法**：多跳问答用 MuSiQue、MultiHop-RAG 这类带多跳标注的基准；检索质量用 RAGAS 框架；本体自身质量看三个指标——实体消解准确率（同一实体几个节点）、关系抽取精度（抽样人审）、查询命中率（业务问题能直接在图上走通的比例）。

**最佳实践六条**：
1. 轻模式起步：先定义 5-10 个核心实体类型与关系，跑通后再加约束。
2. LLM 抽取、人审兜底：置信度低的边进待审队列；每条边带 provenance（出自哪份文档第几段）。
3. 本体即代码：模式层进 git，评审、版本、回滚与代码同流程。
4. 先向量后图：语料先跑 vanilla RAG，用日志统计多跳问题占比，超 30% 再上图。
5. 修正写回：问答中发现错误关系，回写图谱而非只改答案。
6. 查询分类路由（2026 年方向）：事实型走向量，多跳型走图，全局型走社区摘要。

## 九、失败模式与坑

**坑一：本体工程瓶颈。** 症状：建模工作坊开了三个月，类定了两百个，没有一条实例数据。绕法：倒转顺序——先用 LLM 从真实语料抽一版草案图谱，本体从数据里归纳出来，再收敛；本体的第一版应该一周内能写出来。

**坑二：LLM 抽取的幻觉边。** 症状：检索出"某供应商是某客户的子公司"这种看着合理但不存在的关系。怎么发现：抽样人审 + 让每条边强制带出处（provenance），无出处的边检索时降权或过滤。绕法：预定义 schema 约束抽取范围、置信度阈值、关键关系双人审。

**坑三：GraphRAG 成本与延迟失控。** 症状：索引一晚上烧掉几百美元，上线后发现 80% 的查询是简单事实型，根本不需要图。绕法：先按第八节第 4 条统计查询分布；确需全量图的，选 LightRAG 级别成本（约 0.15 美元/文档，官方论文口径）或 HippoRAG（检索零 LLM 调用）。

**坑四：图谱新鲜度漂移。** 症状：源系统客户归属变了三个月，图里还是旧关系，Agent 据此答错。绕法：区分两类更新——事实变更走增量管道（Graphiti 模式，新事件软失效旧边、保留时序），结构变更定期重建索引；给边加有效期字段。

**坑五：把图当银弹。** 症状：为了"知识图谱"这个名头立项，投入半年后发现问答准确率没提升。根因：多数企业查询是语义相似型，本就不该上图。绕法：回到查询类型分布证据（第九节坑三同款方法），用数据决定结构。

## 十、结论与下一步

三行收束：本体给知识定规矩，图谱把规矩填满成数据，GraphRAG 家族让它们能被 LLM 用上；但图只为关系与多跳问题服务，2025 年的反面证据已经把边界画清楚了。

**如果你在写代码**：`pip install lightrag-hku` 一天内跑通，造 50 条多跳问题对比 vanilla RAG，用差异决定投入。
**如果你在架系统**：先向量后图，统计查询分布再定结构；需要"AI 执行业务动作"时研究 Palantir 模式（本体=决策+权限），它和你的 Harness 调研里的 harness 分层正好互补——harness 管执行结构，本体管决策语义。
**如果你是产品经理**：KG 项目卖的不是检索准确率，是决策一致性、影响面分析和审计能力；预算按"模式+数据+运营"三块报，实施周期参考 6-12 周（Palantir 口径），不要按一个搜索功能报。

## 附录：FAQ

**Q1：本体和数据库 schema 有什么区别？**
Schema 约束数据结构（有哪些表哪些列），本体定义语义含义（这个关系允不允许、是否传递、两个概念何时等价）。很多场景 schema 够用；需要跨系统对齐、机器推理、LLM 校验时才需要本体。

**Q2：没有本体可以建知识图谱吗？**
可以，而且很常见——轻模式甚至无模式的属性图大量存在于生产环境。代价：语义靠约定不靠机器校验，跨系统拼接会歧义。实践建议：图谱先行，本体的严格度按需引入。

**Q3：GraphRAG 是不是比 RAG 好？**
不是。2025 年的实证研究显示它在简单查询上可能反输 vanilla RAG，且索引成本高。正确姿势是按需用图：多跳与全局型查询上图，其余保持向量。

**Q4：小团队值得上知识图谱吗？**
看查询类型。如果核心业务问题里多跳关系占比高（影响面、追溯、依赖），一个轻量属性图（LightRAG 量级）一两周能验证价值；否则向量库+元数据过滤足够。

**Q5：知识图谱和向量库会合并吗？**
存储层正在合并：Neo4j 内置向量索引、Neptune Analytics 同时提供 RDF/属性图/向量。但检索语义不会合并——相似度检索与关系遍历回答不同类型的问题，混合路由是确定的终局。

## 参考资料

### 一手（论文 / 标准 / 官方文档 / 仓库）

1. T. Gruber, *A Translation Approach to Portable Ontology Specifications*, Knowledge Acquisition 5(2), 1993 — https://doi.org/10.1006/knac.1993.1009
2. W3C 规范：RDF 1.1 Concepts — https://www.w3.org/TR/rdf11-concepts/ ；OWL 2 — https://www.w3.org/TR/owl2-overview/ ；SHACL — https://www.w3.org/TR/shacl/ ；SPARQL 1.1 — https://www.w3.org/TR/sparql11-overview/
3. ISO/IEC 39075:2024（GQL 标准，2024-04 发布；Neo4j × AWS 联合推动，ISO 页面有反爬，搜标准号即达）
4. Microsoft GraphRAG — arXiv:2404.16130，https://arxiv.org/abs/2404.16130 ；仓库 — https://github.com/microsoft/graphrag
5. LightRAG（HKUDS）— arXiv:2410.05779（EMNLP 2025），https://arxiv.org/abs/2410.05779 ；GitHub — https://github.com/HKUDS/LightRAG
6. HippoRAG（OSU）— arXiv:2405.14831，https://arxiv.org/abs/2405.14831 ；GitHub — https://github.com/OSU-NLP-Group/HippoRAG
7. KAG / OpenSPG（蚂蚁）— arXiv:2409.13731，https://arxiv.org/abs/2409.13731 ；GitHub — https://github.com/OpenSPG/KAG
8. Graphiti（Zep）— arXiv:2501.13956，https://arxiv.org/abs/2501.13956 ；GitHub — https://github.com/getzep/graphiti
9. Mem0g（Mem0）— arXiv:2504.19413，https://arxiv.org/abs/2504.19413
10. Palantir Foundry Ontology 文档 — https://www.palantir.com/docs/foundry/ontology/overview/ ；AIP 文档 — https://www.palantir.com/docs/aip/
11. Neo4j 官方博客：Taxonomy vs. Ontology vs. Knowledge Graph（2026-09）— https://neo4j.com/blog/knowledge-graph/taxonomy-vs-ontology/
12. 行业本体：FIBO — https://spec.edmcouncil.org/fibo/ ；SNOMED CT — https://www.snomed.org/ ；HL7 FHIR — https://hl7.org/fhir/ ；Schema.org — https://schema.org

### 二手（评测 / 分析 / 路线图）

13. Han et al., *RAG vs. GraphRAG 系统性评测* — arXiv:2502.11371，https://arxiv.org/abs/2502.11371 （GraphRAG 简单查询反输：NQ -13.4%、时敏查询 -16.6%）
14. *When to use Graphs in RAG*（GraphRAG-Bench，ICLR 2026）— arXiv:2506.05690，https://arxiv.org/abs/2506.05690 （多跳 +4.5% 但延迟 2.3×）
15. *Use Graph When It Needs: Efficiently and Adaptively Integrating RAG with Graphs* — arXiv:2602.03578，https://arxiv.org/abs/2602.03578
16. Zhou et al., *In-depth Analysis of Graph-based RAG in a Unified Representation*（VLDB 2026, vol.18）— https://www.vldb.org/pvldb/vol18/p5623-zhou.pdf
17. The Data Praxis: *The Knowledge Graph Tooling Landscape*（2026-06，生产级七层栈）— https://www.datapraxis.org/post/knowledge-graph-architecture-2026
18. CopilotKit: *The Five Graphs Every AI Agent Needs*（2026-09，context graph 概念）— https://www.copilotkit.ai/blog/five-graphs-every-ai-agent-needs
19. *Unifying Large Language Models and Knowledge Graphs: A Roadmap*（LLM×KG 路线图）— arXiv:2306.08302，https://arxiv.org/abs/2306.08302
