# AWS Data 与 AI 产品体系调研：三层 Agent 栈、一分为二的 SageMaker 与一个 Iceberg 赌注

> 一句话结论：AWS 的 Data + AI 体系在 2024-2026 年经历了两次结构性重组——AI 侧从"Bedrock 模型托管"扩展为"Bedrock Agents + AgentCore + Q Developer"的三层 Agent 栈，数据侧从 2006-2023 年渐进堆叠的数十个服务重组为"新一代 SageMaker"（SageMaker AI + Unified Studio + Lakehouse + Catalog 四件套），并以 Apache Iceberg 开放表格式为底座把数据仓库、数据湖、AI 资产统一到同一治理平面。与绑定自研模型的云厂商不同，AWS 的 AI 战略锚点是"模型中立托管"：Bedrock 目录同时提供 Anthropic、OpenAI、Google Gemini、Meta、DeepSeek、Qwen 与自家 Nova，把竞争焦点从模型本身转移到运行时、治理和生态集成上。

**TL;DR**
- AI 三层结构：Bedrock（模型 API + 托管 Agents，拿来即用）→ Bedrock AgentCore（2025-10-13 GA，七个模块化运行时服务，自建 Agent 的生产底座）→ Amazon Q Developer（$19/席/月，开发者入口）。三层都模型无关、框架无关
- AgentCore 七件套：Runtime（每会话 Firecracker microVM 隔离）、Memory（短期 $0.25/千事件 + 长期 $0.75/千条）、Gateway（API→MCP 工具，$0.005/千次调用）、Identity、Code Interpreter、Browser、Observability；2026 年新增 Evaluations（GA）、Policy、Harness
- SageMaker 一分为二：原 ML 平台更名 SageMaker AI；新平台 SageMaker Unified Studio（2025-03 GA）把 SQL、Python、可视化 ETL、Data Agent 放进统一 IDE，SageMaker Lakehouse 用 Iceberg 统一 S3/Redshift/外部数据源，SageMaker Catalog 做数据+AI 资产统一治理
- Data 底座 2026 年三大新变量：Aurora DSQL（2025-05 GA，多区域 active-active 分布式 SQL，但每事务 1 万行上限、事务 5 分钟上限、乐观并发控制）、S3 Tables（托管 Iceberg，吞吐为自建 10 倍）、S3 Vectors（S3 原生向量索引，RAG 免外部向量库）
- 规模锚点（Prime Day 2026 官方数据）：DynamoDB 四天 59 万亿请求（峰值 1.92 亿/秒）、Kinesis 峰值 9.88 亿条记录/秒、ElastiCache 日 2.3 千万亿请求
- 商业模式：托管服务按用量计费为主，无"模型-云同体"成本结构；自研 Nova 打性价比档，第三方模型按目录抽成/转售逻辑存在但分成比例不公开

## 一、体系总览：五个平面

| 平面 | 核心服务 | 角色 |
|---|---|---|
| 模型与应用 | Bedrock、Bedrock Agents、Nova、Q Developer、Q Business | 模型 API 市场 + 托管 Agent + 自研模型 + 开发者/企业入口 |
| Agent 运行时 | Bedrock AgentCore（Runtime/Memory/Gateway/Identity/Code Interpreter/Browser/Observability） | 自建 Agent 的生产基础设施 |
| ML 工程 | SageMaker AI（训练/推理/精调）、HyperPod、MLflow | 模型全生命周期，差异化需求（自训、私有部署） |
| 数据+分析平台 | SageMaker Unified Studio、SageMaker Lakehouse、SageMaker Catalog | 统一 IDE + Iceberg 湖仓 + 统一治理 |
| 数据底座 | S3（Tables/Vectors）、Redshift、Glue、Athena、EMR、Kinesis、DynamoDB、Aurora（含 DSQL）、Timestream、Neptune、DocumentDB、Keyspaces、MemoryDB、MSK | 15+ 专用引擎，按工作负载选型 |

理解 AWS 的关键：它不是一条产品线的演进，而是**两条哲学在同一底座上的叠加**——"15+ 专用数据库"哲学（每个工作负载一个最优引擎）和"统一治理"哲学（Iceberg + Catalog 把一切拉回同一平面）。 tension 由此产生，下文多处可见。

## 二、AI 应用层：三层 Agent 栈

**第一层：Bedrock 模型托管**（2023-09 GA）。模型目录是其核心资产：Anthropic Claude、OpenAI、Google Gemini、Meta Llama、Mistral、DeepSeek、Qwen、AI21、Cohere、Stability、Writer + 自家 Nova 系列。单一 API、单一鉴权、统一 Guardrails。配套能力：Knowledge Bases（托管 RAG）、Guardrails（含 Automated Reasoning checks——用形式逻辑校验防止事实性错误，为 AWS 独有）、模型评估、批量推理与蒸馏。

**第二层：Bedrock Agents → AgentCore**。旧的 Bedrock Agents 是"托管黑盒"：给个模型+API 清单+数据，AWS 帮你拼好 Agent。AgentCore（re:Invent 2024 发布，2025-10-13 GA）是更底层的模块化平台，明确模型无关（Bedrock 内外均可）、框架无关（Strands、LangGraph、CrewAI、LlamaIndex、Google ADK、OpenAI Agents SDK 都直接跑）。定价按用量：Runtime $0.0895/vCPU-小时 + $0.00945/GB-小时，空闲近零成本；Memory 按事件与存储计费；Gateway 把存量 API/Lambda 一键变成 MCP 工具（$0.005/千次调用）。2026 年持续加模块：Evaluations GA（2026-03-31）、托管 Harness（2026-04-22）、Policy、Managed Knowledge Base、Web Search。隔离是卖点：每个用户会话运行在独立 Firecracker microVM，会话结束内存即擦除。

**第三层：Amazon Q Developer**。$19/席/月，从代码补全升级为 agentic chat：/dev（自然语言跨文件实现功能）、/transform（Java 8→21、.NET Framework→9、COBOL 迁移）、/test、/doc。与 AgentCore 的分界：Q 是"工程师的 AI"，AgentCore 是"你产品的 AI"。

**自研模型 Nova 家族**定位清晰：Micro/Lite/Pro/Premier 语言梯度，Sonic（语音）、Canvas（图像）、Reel（视频）、Act（GUI 操作）补全模态。不追求榜单第一，打性价比与企业合规场景；头部能力依赖目录里的 Anthropic/OpenAI。

## 三、ML 工程层：SageMaker 的"一拆为二"

SageMaker 2017 年发布时是托管训练/部署服务，五年加 380+ 功能后膨胀难用。2024-12-03 re:Invent 宣布重组：

- **SageMaker AI** = 原服务更名续存：训练、推理、精调、JumpStart、Canvas、Feature Store、Clarify、Pipelines、HyperPod（大规模训练集群，2025 年加 checkpointless 弹性训练）
- **SageMaker Unified Studio**（2025-03-13 GA）= 统一 IDE：SQL 编辑器、Python notebook、可视化 ETL、自然语言 Data Agent 共处一室，底层引擎可选 EMR/Athena/Glue
- **SageMaker Lakehouse** = 开放湖仓：Apache Iceberg 为表格式，统一 S3、Redshift、第三方存储（含 Snowflake），15+ zero-ETL 集成
- **SageMaker Catalog** = 数据+AI 资产统一治理：血缘、权限、语义搜索、敏感检测，集成 Bedrock Guardrails

re:Invent 2025 再加：serverless 模型定制（无服务器精调 Nova 等）、托管 MLflow、agentic notebooks、SageMaker Data Agent、非结构化数据入目录（S3 资产元数据化）。案例口径：Carrier 基于该栈建 Iceberg 湖仓，实施成本降 66%、自然语言转 SQL 准确率升 38%。

## 四、数据底座：专用引擎 + 三个 2025-2026 新变量

**经典阵容**（按工作负载分库）：Redshift（数仓）、DynamoDB（键值 NoSQL）、Aurora/RDS（关系型）、DocumentDB（文档）、Keyspaces（Cassandra）、Neptune（图）、Timestream（时序）、MemoryDB/ElastiCache（内存）、OpenSearch（搜索）、QLDB（账本）。流与集成：Kinesis、MSK（Kafka）、DMS、Glue（ETL+目录+质量）、Step Functions 编排。查询：Athena（S3 serverless SQL）、EMR。BI：QuickSight。治理：Lake Formation（细粒度授权）、DataZone（数据产品目录与共享，与 Glue 数据质量集成）。

**新变量一：Aurora DSQL**。AWS 史上野心最大的数据库：serverless、PostgreSQL 兼容、多区域 active-active、五个 9 可用性、空闲计费归零。re:Invent 2024 preview，2025-05-27 GA。但限制清单必须正视：PG 功能子集（无外键、触发器、视图、存储过程、PostGIS）、单事务 1 万行上限、事务 5 分钟/连接 1 小时超时、乐观并发控制在热点行写冲突下会产生重试风暴、单内置库。定位是"全球 OLTP 新应用"，不是现有 PG 应用的平移。

**新变量二：S3 Tables**。为 Iceberg 表专门设计的托管存储：吞吐约为普通 S3 上自建 Iceberg 的 10 倍，表级安全、自动压缩与垃圾回收，Lake Formation 授权直通。

**新变量三：S3 Vectors**。S3 原生向量索引，免外部向量库即可对 embedding 做相似度检索——与 Bedrock 组合成"无第三方组件"的 RAG 管线。加上 Aurora PostgreSQL 的 pgvector 与 Neptune Analytics 的图+向量混合检索，向量能力被系统性"内化"到存量存储里。

**AI 融合动作**：Glue 零 ETL 打通 Aurora→Redshift；DynamoDB 推出 API 兼容的开源适配器 ExtendDB（2026-05）；DataZone 吸收 Glue 数据质量指标；统一特征是"让 AI 消费数据时不搬数据"。

## 五、定价与商业模式

- **模型层**：按 token 计费，第三方模型价格与厂商官方价基本一致（如 DeepSeek 同价），AWS 赚托管与集成溢价；Nova 显著低于头部第三方（2024-05 发布时主力模型 $0.0008/千 tokens 的价位策略曾引发行业降价潮）
- **AgentCore**：纯按用量，无预留实例；空闲归零的设计对突发型 Agent 负载友好
- **Q Developer**：$19/席/月，席位制，与 Copilot 同价带
- **数据层**：存量服务成熟定价（Redshift Serverless RPU 小时、DynamoDB 读写单元、Kinesis 分片时、Glue DPU）；新服务（DSQL、S3 Tables/Vectors）按 serverless 用量计费
- 商业模式的本质：**没有"模型-云同体"的内部负载摊薄**，AWS 靠规模采购、专用硬件（Trainium/Inferentia）和托管溢价维持价格竞争力；客户议价杠杆来自用量承诺（Savings Plans/EDP）而非厂商补贴

## 六、宣称 vs 可验证事实

| 宣称 | 可验证性 | 评估 |
|---|---|---|
| S3 上已建 1000 万+ 数据湖 | AWS re:Invent 2025 官方口径 | 方向可信，无法独立审计 |
| S3 Tables 吞吐为自建 Iceberg 10 倍 | 官方基准 | 官方基准通常选优条件，需自有负载压测 |
| Carrier 成本降 66%、NL2SQL 准确率 +38% | re:Invent 客户案例 | 单客户案例，非中位数 |
| Prime Day 2026：DynamoDB 59 万亿请求/4 天、Kinesis 峰值 9.88 亿条/秒 | AWS 官方博客（2026-10-07） | 自家系统数据，量级可信；证明的是自家电商负载，非客户负载 |
| AgentCore "模型无关" | 技术文档 + 第三方实测（2026-10 评测） | 成立，但外部模型走直连 API，不享受 Bedrock 统一计费/Guardrails 全链 |
| Bedrock Guardrails Automated Reasoning 为独有 | 2026 年多平台对比评测 | 目前成立（形式逻辑校验无对标） |

## 七、优势拆解（结构性）

1. **模型中立是唯一认真执行的战略**。Azure 押 OpenAI、Google 押 Gemini，只有 AWS 把 Anthropic/OpenAI/Gemini/Llama/DeepSeek/Qwen 放进同一目录同一套 IAM——客户可以逐模型选型，锁定的是 AWS 平台而非某个模型厂商。
2. **Agent 栈的纵深**。从"托管黑盒"（Bedrock Agents）到"生产运行时"（AgentCore 七件套）到"开发入口"（Q Developer）到"开源框架"（Strands SDK）四层齐备，且 AgentCore 的微 VM 会话隔离是当前市场上最严格的 Agent 沙箱叙事。
3. **Iceberg 赌注的防守价值**。S3 Tables + Lakehouse + Catalog 把开放表格式做成默认路径：数据留在 S3、引擎可换（包括 Snowflake/Databricks 都支持 Iceberg），把"逃离 AWS"的成本降到理论最低——以此换取"默认留在 AWS"。这是用开放性对抗专用数仓锁定。
4. **专用数据库的宽度**。15+ 引擎按工作负载覆盖之全，业内无第二家；Aurora DSQL 补上"全球分布式 SQL"最后一块拼图（此前这块只有 Google Spanner 和 CockroachDB）。
5. **企业合规纵深**。AgentCore 覆盖 15 区域含 GovCloud；FedRAMP High、HIPAA、ISO、SOC、CSA STAR Level 2 全套；Guardrails 自动化推理校验独一份。

## 八、主打场景与非目标场景

**场景一：多模型策略的企业 AI 平台**。需要在 Claude/GPT/Gemini/Llama 间动态选路、统一审计与成本归集的团队，Bedrock 是当前唯一认真的选项。

**场景二：自建 Agent 产品化**。已有 LangGraph/CrewAI 原型，需要会话隔离、持久记忆、MCP 工具治理、身份与审计的生产化——AgentCore 的模块化允许按需采用（Runtime+Memory 起步最常见）。

**场景三：存量 AWS 数据的 AI 化**。数据已在 S3/Redshift/DynamoDB，用 Knowledge Bases + S3 Vectors + 零 ETL 把 RAG 管线搭在原地，不搬数据。

**场景四：全球分布式新 OLTP 应用**。跨区域 active-active、PG 语法、serverless 弹性——DSQL 的靶心（接受其功能子集约束）。

**非目标场景**：
- 单模型深度绑定某厂商（如全押 OpenAI）：Bedrock 的中立性反而是溢价
- 非 AWS 为主的异构云：数据面重力全在 AWS，跨云联邦治理能力弱于宣传
- 强 OLAP 实时分析：Redshift/Athena 组合成熟但非性能标杆，ClickHouse/BigQuery 各有所长
- DSQL 约束敏感型 PG 存量应用（见第四节限制清单）
- 预算极敏感的小团队：免费额度与低价区更友好的是别家

## 九、同类坐标

| 厂商 | AI 结构 | 数据结构 | 核心差异 |
|---|---|---|---|
| AWS | Bedrock（模型中立目录）+ AgentCore + Q + Nova | 15+ 专用引擎 + Iceberg 统一治理 | 模型中立 + 引擎宽度 + 开放格式赌注 |
| 微软 Azure | Foundry（OpenAI 深度绑定）+ Copilot 全家桶 | Fabric 一站式 + OneLake | 绑定 OpenAI，卖 M365 集成 |
| Google Cloud | Vertex AI + Gemini 全家桶 | BigQuery 单引擎统治 + Spanner | 自研模型最强，数仓性能标杆 |
| 火山引擎 | 豆包自研家族 + 方舟 + Agent 栈 | ByteHouse/DataLeap/VeDI | 模型-云-内部负载同体的成本结构 |

## 十、判断与建议

1. AWS 战略的可读性在增强：它承认模型层会商品化，把赌注下在**运行时（AgentCore）、治理（Catalog/Guardrails）、开放格式（Iceberg）**三层。买 AWS 的 AI，买的不是模型而是"模型可以被替换"的权利。
2. 选 Bedrock 的企业要算清两本账：多模型自由 vs 单一模型厂商的深度优化（功能首发、批量折扣通常向绑定方倾斜）；以及第三方模型经 Bedrock 转售时的支持响应链路。
3. AgentCore 评估时优先验证四件事：会话冷启动延迟（microVM 换隔离的代价）、Memory 事件计费的实际账单形状、Gateway 对存量内部 API 的认证覆盖、Guardrails-in-policy 对工具调用的拦截性能。
4. DSQL 按"新应用默认选项"评估，不要按"Aurora 平替"评估；把它放进选型清单的同时，把功能子集核对表写进迁移评估。
5. 观察点：SageMaker Unified Studio 能否真的吃掉 Glue/EMR/Athena 的分散体验（AWS 历史上"统一层"成功率不高）；AgentCore 与 LangChain 等框架的商业关系（平台层与框架层的利益分配）；Nova 在 Coding/Agent 基准上能否从"性价比档"挤进"能力档"。

## 参考资料

- [AWS Bedrock AgentCore vs Azure Foundry vs Vertex AI 三方对比（tech-insider，2026-10-09，含定价与区域核对）](https://tech-insider.org/aws-bedrock-agentcore-vs-azure-foundry-vs-vertex-ai-agent-builder-2026/)
- [Pingax：AgentCore 完整指南（2026-07-09，组件与 Memory/DynamoDB 辨析）](https://pingax.com/aws-bedrock-agentcore-the-complete-2025-guide/)
- [ExploreAgentic：SageMaker vs Bedrock Agent 优先决策框架（2026-07-03，混合架构模式）](https://www.exploreagentic.ai/insights/bedrock-vs-sagemaker/)
- [Kosmoy vs AgentCore（2026-08-16，隔离、Guardrails、合规纵深对比）](https://www.kosmoy.com/resources/blog/kosmoy-vs-aws-bedrock-agentcore/)
- [CloudAQube：AgentCore 部署指南与 AIP-C01 考试映射（2026-06-26）](https://cloudaqube.com/blog/aws-bedrock-agentcore-guide-2026)
- [hidekazu-konishi：SageMaker 历史时间线（2026-05-31，2017-2025 重组全录）](https://hidekazu-konishi.com/entry/aws_history_and_timeline_amazon_sagemaker.html)
- [zenn：re:Invent 2025 ANT351 讲座转录——SageMaker 统一数据与 AI 平台（Carrier 案例）](https://zenn.dev/kiiwami/articles/e10e72fc74aeedbe?locale=en)
- [AntStack：ANT216 re:Invent 2025 SageMaker 新功能汇总](https://www.antstack.com/talks/reinvent25/aws-re-invent-2025---whats-new-with-amazon-sagemaker-in-the-era-of-unified-data-and-ai-ant216/)
- [AWS 官方博客：Prime Day 2026 数字（DynamoDB/Kinesis/Aurora 峰值，2026-10-07）](https://aws.amazon.com/blogs/aws/all-the-numbers-amazon-prime-day-2026-powered-by-aws/)
- [Cloudvisor：Aurora DSQL 详解与限制清单（2026-05-10）](https://cloudvisor.co/amazon-aurora-dsql/)
- [ThinkMove：AWS 数据库服务完整指南（2026-05-12，15+ 引擎与 AI 化趋势）](https://thinkmovesolutions.com/blogs/complete-guide-to-aws-database/)
- [sailor.sh：RDS/DynamoDB/Aurora/Redshift/ElastiCache 认证向指南（2026-03-22）](https://sailor.sh/blog/aws-database-services-certification-guide/)
- [classmethod：2026-05 AWS 趋势（ExtendDB、Aurora DSQL CDC、Aurora MySQL 8.4 GA）](https://dev.classmethod.jp/articles/aws-trendcheck-202605/)
- [examcert.app：AgentCore vs Q Developer 选型（2026-05-05）](https://www.examcert.app/blog/aws-bedrock-agentcore-vs-amazon-q-developer-2026/)

---

*数据截至 2026-10-21。服务可用区域与价格以 AWS 官网实时信息为准；Prime Day 数据为 AWS 官方博客口径。*
