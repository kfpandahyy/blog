# 阿里云 AI + Data 产品体系全景：一张图看懂 60+ 产品及关系

> 一句话结论：阿里云是国内唯一"从芯片到应用入口"全链路自研的云厂商，AI 与数据两条产品线在 2026 年已经合并成一张图。本文上半部分用一张图说清结构，下半部分对模型平台与数据产品做深度拆解。

**TL;DR**
- 六层结构：L0 芯片（平头哥）→ L1 智算 → L2 模型（Qwen）→ L3 平台 → L4 数据 → L5 应用入口
- 三条流向：训练流（数据→算力→模型）、服务流（平台调模型 + 取上下文）、回流（Agent 数据写回 L4，数据产品变成"智能原料厂"）
- 战略骨架：吴泳铭定调"AI 模型、AI 芯片、AI 云"三基石；云栖 2026 发布 Agentic Cloud（32+ 新品）；百炼升级为"千问 AI 平台"统一出口
- 2026 年最大变量在 Data 侧：PolarDB 一年发布 9+ 个 Agent 相关产品，MaxCompute / DataWorks 长出 Data Agent，Tair KVCache 反哺推理

## 一、总图：六层结构

```
L5  应用入口    千问App · 千问办公 · 钉钉AI · 夸克 · QwenNote · 手机/车机(Qwen Intelligence)
L4  数据产品    瑶池数据库(PolarDB/RDS/PolarDB-X/AnalyticDB/Lindorm/Tair) · MaxCompute · Hologres · DataWorks · 向量检索全家桶
L3  平台与运行时 千问AI平台(原百炼) · PAI · AgentCore · Agent Sandbox · 无影AgentBay · 魔搭ModelScope
L2  模型        Qwen 家族(Max/Next/Coder/VL/Omni/Audio/Image/Wan视频/百聆)
L1  智算基建    灵骏集群 · 磐久超节点 · HPN网络 · CPFS · OSS · 飞天+CIPU
L0  芯片        真武V900/M890(AI) · 倚天(CPU) · ICN Switch · 磐脉(网卡) · 镇岳(存储主控)
```

层间关系一句话：**L0→L2 解决"智能怎么生产"，L3 解决"智能怎么编排"，L4 解决"智能吃什么数据"，L5 解决"谁在用"**。

## 二、分层清单

> 官网地址均于 2026-10-24 逐一验证可达；「—」表示官方无独立产品页（底座/架构类能力）。

### L0 芯片（平头哥）

| 序号 | 产品名称 | 产品介绍 | 官网地址 |
|---|---|---|---|
| 1 | 真武 V900 | 新一代训推一体 AI 芯片，性能为 M890 三倍，2027 Q1 量产 | [平头哥官网](https://www.t-head.cn) |
| 2 | 倚天 710 | 自研服务器 CPU | [平头哥官网](https://www.t-head.cn) |
| 3 | ICN Switch | 自研互联芯片，千卡全带宽互联 | [平头哥官网](https://www.t-head.cn) |
| 4 | 磐脉 | 自研智能网卡 | [平头哥官网](https://www.t-head.cn) |
| 5 | 镇岳 | 自研 SSD 存储主控芯片 | [平头哥官网](https://www.t-head.cn) |

意义不在单品性能，而在"芯片—集群—模型"可以协同设计：Qwen3.8-Max 已演示仅凭一份总线规范自主完成芯片物理实现优化，面积减 42%。

### L1 智算基础设施

| 序号 | 产品名称 | 产品介绍 | 官网地址 |
|---|---|---|---|
| 1 | 灵骏 | 智算集群品牌；结合 V900 超节点与新网络架构，单一集群可扩至 50 万卡（2026 云栖官方口径） | [产品页](https://www.aliyun.com/product/lingjun) |
| 2 | 磐久超节点 | 集成 V900 + ICN Switch + 磐脉 + 镇岳的整机柜 | [随灵骏售卖](https://www.aliyun.com/product/lingjun) |
| 3 | HPN 8.0 | 数据中心网络架构 | —（网络架构，非独立售卖产品） |
| 4 | CPFS | 高性能并行文件存储（训练用），2026 云栖发布新一代 | [产品页](https://www.aliyun.com/product/cpfs) |
| 5 | OSS | 对象存储，兼作向量数据底座（Vector Bucket） | [产品页](https://www.aliyun.com/product/oss) |
| 6 | 飞天 + CIPU | 自研云操作系统 + 云基础设施处理器 | —（自研底座） |

### L2 模型（Qwen / 通义家族）

| 序号 | 产品名称 | 产品介绍 | 官网地址 |
|---|---|---|---|
| 1 | Qwen3-Max | 旗舰，万亿参数、36T tokens 预训练 | [Qwen 官网](https://qwen.ai) |
| 2 | Qwen3.8-Max | 已做到 33 轮零人工自迭代训练（自主搭流程、造数据、定位缺陷） | [Qwen 官网](https://qwen.ai) |
| 3 | Qwen3-Next | 效率款，80B 总参 / 3B 激活，训练成本降 90% | [Qwen 官网](https://qwen.ai) |
| 4 | Qwen3-Coder | 代码模型，与 Claude Code 系统联合训练 | [Qwen 官网](https://qwen.ai) |
| 5 | Qwen3-VL | 视觉 Agent 模型 | [Qwen 官网](https://qwen.ai) |
| 6 | Qwen3-Omni | 全模态实时交互、MCP 编排 | [Qwen 官网](https://qwen.ai) |
| 7 | Qwen-Audio-3.1 | 语音家族：ASR / TTS / Realtime / LiveTranslate 同传 | [Qwen 官网](https://qwen.ai) |
| 8 | Qwen-Image-3.1 | 图像生成 | [Qwen 官网](https://qwen.ai) |
| 9 | Wan 万相 | 视频生成，2026-11 发下一代新版 | [万相官网](https://tongyi.aliyun.com/wanxiang) |
| 10 | 通义百聆 | 语音开源模型，下载 5.6 亿+ | [通义官网](https://tongyi.aliyun.com) |
| 11 | 魔搭 ModelScope | 模型社区：17 万+ 模型、2500 万开发者、9200+ MCP Server，衍生模型 3.4 万+ | [modelscope.cn](https://modelscope.cn) |

Qwen4 在训（官方口径：后续 Qwen4.5 / Qwen5 将扩展至 5-10T 参数）。

### L3 平台与 Agent 运行时

| 序号 | 产品名称 | 产品介绍 | 官网地址 |
|---|---|---|---|
| 1 | 千问AI平台（原百炼） | 模型服务 + Agent Studio（六环节生产级 Agent）+ Agent API（官方计划 2026-10 发布）；客户数年增 6 倍 | [产品页](https://www.aliyun.com/product/bailian) |
| 2 | PAI | 机器学习平台（训练/微调/部署全流程） | [产品页](https://www.aliyun.com/product/bigdata/learn) |
| 3 | AgentCore | Agent 构建与治理平台（2026 云栖新品） | [产品页](https://www.aliyun.com/product/agentcore) |
| 4 | Agent Sandbox | Agent 隔离运行环境（2026 云栖新品） | [随 AgentCore](https://www.aliyun.com/product/agentcore) |
| 5 | 无影 AgentBay | 云端桌面/浏览器，Agent 的操作界面 | [产品页](https://www.aliyun.com/product/ecs/wuying) |
| 6 | ADK / ADP | Agent 开发框架（代码）/ 低代码平台（20 万开发者、80 万+ Agent）；7 大企业级 Server 含支付宝 Pay Server | [随百炼](https://www.aliyun.com/product/bailian) |

### L4 数据产品（全家桶的主体）

| 序号 | 产品名称 | 产品介绍 | 官网地址 |
|---|---|---|---|
| 1 | PolarDB | 云原生关系库；AI 化最激进的一支（Agentic Database、Agent Memory、LakeBase），见第五节 | [产品页](https://www.aliyun.com/product/polardb) |
| 2 | PolarDB-X | 分布式事务库（TPC-C 双榜认证） | [产品页](https://www.aliyun.com/product/drds)（旧域名 drds） |
| 3 | RDS | 通用托管 MySQL/PG，入门首选 | [产品页](https://www.aliyun.com/product/rds) |
| 4 | AnalyticDB | 云数据仓库（PG 兼容），向量引擎 + NL2SQL | [产品页](https://www.aliyun.com/product/analyticdb) |
| 5 | Lindorm | 多模融合（宽表/时序/搜索），IoT 万亿行 | [产品页](https://www.aliyun.com/product/lindorm) |
| 6 | Tair | 内存库（Redis 兼容）；KVCache 下沉给 LLM 推理破显存墙 | [产品页](https://www.aliyun.com/product/kvstore) |
| 7 | DAS | 跨库自治诊断调优 | [文档页](https://help.aliyun.com/zh/das/) |
| 8 | MaxCompute | 离线数仓，双 11 峰值验证 | [产品页](https://www.aliyun.com/product/odps) |
| 9 | Hologres | 实时数仓（PG 兼容），与 MaxCompute 湖仓双引擎 | [产品页](https://www.aliyun.com/product/hologres) |
| 10 | DataWorks | 一站式数据开发治理（50+ 数据源、字段级血缘），2026 全面 Agent 化 | [产品页](https://www.aliyun.com/product/bigdata/ide) |
| 11 | EMR | 开源大数据栈（Hadoop/Spark） | [产品页](https://www.aliyun.com/product/bigdata/emapreduce) |
| 12 | Flink 实时计算 | 实时计算引擎云服务 | [产品页](https://www.aliyun.com/product/bigdata/sc) |
| 13 | SLS | 日志服务，Agent 行为日志归宿之一 | [产品页](https://www.aliyun.com/product/sls) |
| 14 | Quick BI | BI 报表 | [文档页](https://help.aliyun.com/zh/quickbi/) |
| 15 | OpenSearch | 开放搜索 | [产品页](https://www.aliyun.com/product/opensearch) |
| 16 | DataV | 数据可视化大屏 | [产品页](https://www.aliyun.com/product/bigdata/datav) |
| 17 | DashVector | 独立向量检索服务 | [文档页](https://help.aliyun.com/zh/dashvector/) |
| 18 | Tablestore | 表格存储，Agent 记忆/知识库场景 | [产品页](https://www.aliyun.com/product/ots) |
| 19 | OSS Vector Bucket | 向量数据存 OSS，成本较自建开源库降 95% | [随 OSS](https://www.aliyun.com/product/oss) |

选型口碑（51CTO 2026-09 行业适配指南）：电商 PolarDB+Tair+AnalyticDB，金融 PolarDB-X，IoT Lindorm；RDS→PolarDB→PolarDB-X 平滑升级。向量检索策略：**不做独立向量库单品，让每个存储都长向量能力**（库内向量见 PolarDB/AnalyticDB/Lindorm/Tair 各行），数据在哪向量就在哪；代价是选型复杂度高。

### L5 应用与入口

| 序号 | 产品名称 | 产品介绍 | 官网地址 |
|---|---|---|---|
| 1 | 千问 App | C 端助手 | [通义千问](https://www.tongyi.com) |
| 2 | 千问办公 | 企业 Agent 平台 + 企业上下文（Enterprise Context）；配 QwenNote 随身硬件 | [通义千问](https://www.tongyi.com)（千问系统一入口） |
| 3 | 钉钉 AI | 办公入口，数字员工与人类员工在 IM 协同 | [钉钉官网](https://www.dingtalk.com) |
| 4 | 夸克 | 搜索/学习入口 | [夸克官网](https://www.quark.cn) |
| 5 | QwenNote | 随身硬件，外接千问办公干活（2026 云栖新品，暂无独立产品页） | [通义千问](https://www.tongyi.com) |
| 6 | Qwen Intelligence | 手机/车机端侧 Agent 全栈方案（荣耀 Magic9 首发，规划/操作/创作三套模型） | [通义千问](https://www.tongyi.com) |


## 三、产品之间的关系（上半部分的图，落到地上）

三条流向，看清就理解了整家公司：

**1. 训练流（自下而上，生产智能）**
业务数据（L4：MaxCompute 数仓、OSS 语料）→ 灵骏集群（L1）→ Qwen（L2）。芯片（L0）决定这条链的成本上限。魔搭把训好的模型分发给社区，反哺 Qwen 的生态位。

**2. 服务流（自上而下，消费智能）**
用户（L5）→ 千问AI平台 / AgentCore（L3）→ 运行时调用 Qwen（L2）+ 从 L4 取上下文与记忆。**L3 是中枢**：左手调模型、右手查数据，Agent 的编排、工具、状态问题都在这一层解决。

**3. 回流（闭环，数据越用越厚）**
Agent 和应用的运行数据写回 L4：行为日志进 Lindorm/SLS，业务结果进 PolarDB，治理后沉淀为企业上下文再喂给千问办公。**L4 因此从"成本中心"变成"智能的原料厂"**——这是阿里数据产品线在 AI 时代的真实位置。

四个关键耦合点：

- **Tair × 推理**：数据库技术反哺 AI。KVCache 从显存下沉到分布式内存（GTC 2026 亮相），直接降推理成本、破显存墙
- **PolarDB × Agent**：AI 反哺数据库。Branch 穿梭 = Agent 试错隔离，Agent Memory = 上下文持久化，MCP/CLI 原生交互 = Agent 直接驱动数据库
- **DataWorks × 千问办公**：数据治理成果（血缘、口径、元数据）是企业上下文的前置输入
- **魔搭 × 千问AI平台**：开源获客、平台变现的分工闭环

## 四、深挖一：模型与平台层

**为什么模型栈值得单独看**：阿里是国内唯一把"开源模型"当战略武器用的云厂商。Qwen 锁定全球开发者心智（衍生 3.4 万+），云与数据产品承接生产负载，"开源获客、全栈变现"。

平台层的排布逻辑：PAI 管"训出来"，千问AI平台管"用起来"，AgentCore/Sandbox 管"跑起来"，无影管"看得见"。Agent Studio 的六环节（开发/运行/上下文/工具/评测进化/安全）对应企业级 Agent 落地的完整生命周期，其中"上下文"环节直接吃 L4 数据产品——这是平台层与数据层的官方接缝。

## 五、深挖二：数据产品线（本次全景的重头）

### 5.1 瑶池六线的分工

选型口碑（51CTO 2026-09 行业适配指南）：电商 PolarDB+Tair+AnalyticDB，金融 PolarDB-X，IoT Lindorm；RDS→PolarDB→PolarDB-X 是平滑升级路径。

### 5.2 PolarDB 的"Agent 全家桶"（一年 9+ 个产品的演进链）

| 时间 | 产品/能力 | 实质 |
|---|---|---|
| 2023-09 | PolarDB for AI、AAT 增强表、集成千问 7B | SQL 里调模型、推理结果写回表 |
| 2026-01 | **AI 数据湖库 Lakebase**（湖库一体、全模态一致存取）、模型算子化、AI 节点 VNode（Serverless GPU 入 K8s） | "AI 就绪数据库"四大支柱落地（李飞飞在 PolarDB 开发者大会定调） |
| 2026-01 | **PolarDB Agent Memory**（100% 兼容开源 Mem0） | Agent 长效记忆托管 |
| 2026-02 | PolarSearch（PolarDB 数据自动向量化同步） | 库内语义检索 |
| 2026-03 | **PolarDB Agent Express**：基于开源 AI 助理 OpenClaw 构建的企业级 PaaS | 企业私有化部署 AI 助理 |
| 2026-03 | Polar Agent Memory 长期记忆引擎（向量+知识图谱双模） | 跨会话个性化记忆 |
| 2026-03/05 | **PolarDB Agent LakeBase**：百万级 Agent 的持久化 Workspace（快照、目录配额/克隆/共享） | Agent 运行基座 |
| 2026-06 | 企业知识空间、PolarDB Agentic DataBase（MCP 接入、QoderWork 接入） | 知识与算力收口 |
| 2026-07 | **PolarDB MySQL Agentic Database 正式形态**：Scale to Zero、秒级 Branch 分支穿梭、库内多模检索推理、MCP/CLI 原生交互 | 面向海量 Agent 场景的数据库品类定义 |

读法：**PolarDB 在把"数据库"重新定义为"Agent 的运行时与状态层"**。这与 AWS Aurora DSQL/Durable Execution 思路异曲同工，但阿里把记忆、Workspace、知识空间一并打包，产品化速度更快——Agent Express 直接拥抱开源生态而非纯自研。

### 5.3 大数据体系的 Agent 化

- **DataWorks 智能化四连**：Data Agent（自然语言驱动集成/开发/数据地图/治理）、数据运维 Agent（多维诊断 + 对话框执行运维）、SQL 事前检查（质量管控前移到编码时）、离线同步内嵌大模型（同步从"搬运"变"加工"，PrivateLink 私网不出域）
- **MaxCompute 多源 DataAgent**（2026-10-10 上线）：一个 AI Query 联合分析 MaxCompute 离线数仓、Hologres/StarRocks 实时数据、Milvus 向量、OSS 文件和本地资料
- **边界收缩信号**（2026-02-26）：MaxCompute 关闭 DataWorks 数据湖集成入口，湖仓一体重心压到 MaxCompute+Hologres 主线——产品边界在收敛，不是扩张

### 5.4 向量与检索：全家桶式打法

见第二节 L4。一句话评价：阿里不做"独立向量库"赌博，让每个存储产品都长出向量能力。风险是选型复杂度高，好处是数据在哪向量就在哪，不用搬迁。

## 六、场景选型速查

| 我要做 | 直接看这套组合 |
|---|---|
| 企业级 Agent | 千问AI平台（Agent Studio）+ PolarDB Agentic Database/Tablestore（状态与记忆） |
| 自然语言取数 | DataWorks Data Agent + MaxCompute 多源 DataAgent |
| RAG / 语义检索 | 数据在哪就用哪个库的向量能力；独立场景 DashVector |
| 模型微调/训练 | PAI + 灵骏；开源模型从魔搭拉 |
| 端侧 Agent（手机/车） | Qwen Intelligence 整套方案 |

## 七、优势、竞争与风险

**结构性优势**：
1. 全栈唯一性：芯片 + 十万卡集群 + 开源模型 + 双开发平台 + C/B 端入口 + 全品类数据产品，国内只有华为部分可比，**数据侧完整度阿里第一**
2. 开源即战略：Qwen 生态锁定开发者，全栈承接负载
3. Data+AI 双向反哺：Tair 给推理加速，PolarDB 给 Agent 供状态层——"AI 超级计算机"叙事在数据层的兑现
4. 真实场景验证：双 11 锤炼的调度与并发，金融核心迁移案例，企业级信任资产
5. 商业闭环最长：模型 API → 云资源 → 数据平台 → 支付（Pay Server）→ 交易（ACT 协议）→ 终端

**竞争对位**：

| 维度 | 阿里 | 对手 |
|---|---|---|
| 大数据平台 | MaxCompute+DataWorks（双 11 背书） | 腾讯云 WeData、火山引擎、华为 MRS |
| 数据库 | 瑶池六线 | 华为 GaussDB、腾讯 TDSQL、达梦/信创系 |
| AI+DB 融合 | PolarDB Agent 全家桶、Tair KVCache | AWS Aurora DSQL/Durable Execution、Neon/Supabase（开发者心智） |
| 模型+开源 | Qwen | DeepSeek、字节豆包 |

**风险三条**：一是应用层组织动荡（千问办公为三合一重组产物）；二是数据产品 AI 化快但单价承压，Agentic Database 的 Scale to Zero 意味着收入弹性绑定客户成功；三是生态绑定双刃剑——DataWorks 仅公有云、治理成果难迁出，多云时代是结构性短板。

## 八、判断

1. 阿里云的真实版图是"AI 栈 + Data 栈"双栈咬合：AI 栈生产智能，Data 栈供给上下文与承接状态。2026 年两者的接缝处（PolarDB Agentic Database、MaxCompute DataAgent、DataWorks Data Agent、Tair KVCache）是整家公司产品创新密度最高的区域。
2. 与 AWS 对照：AWS 走托管网关 + 模型中立，靠 Durable Execution/Aurora DSQL 定义 Agent 状态层；阿里走开源模型自营 + 数据产品全家桶 Agent 化。阿里的产品化速度更快，但锁定更深。
3. 给从业者的实用建议：已上阿里云的企业，Agent 数据底座直接评估 PolarDB Agentic Database + Tablestore 组合，不必自建记忆/Workspace 层；数据团队把 DataWorks Data Agent 和 MaxCompute 多源 DataAgent 当"自然语言取数"入口试点，但治理规则与口径仍需人工确认。

## 参考资料

- [2026 云栖大会首日发布合集（真武V900 / AgentCore / CPFS / 企业上下文），DoNews，2026-09-22](https://www.donews.com/news/detail/1/6720181.html)
- [财新：阿里发布智能体电脑等多款产品，2026-09-23](https://www.caixin.com/2026-09-23/102488138.html)
- [新华网：阿里云升级全栈 AI 体系，2025 云栖大会技术发布，2025-09-24](http://www.news.cn/tech/20250924/7f3377e5e81441eb95bc3280539b3594/c.html)
- [InfoQ：2026 年的云栖，阿里给 AI 行业递了一把新尺子，2026-09-23](https://www.infoq.cn/article/zSPQbRwmS0AGpeTAhu9r)
- [观察者网：阿里云 PolarDB 发布全面内化的 AI 能力，2026-01-20](https://www.guancha.cn/economy/2026_01_20_804479.shtml)
- [阿里云文档：PolarDB MySQL 控制台发布记录（Agent 产品全时间线，2026-10-07 抓取）](https://help.aliyun.com/zh/polardb/polardb-for-mysql/release-notes)
- [阿里云文档：MaxCompute 2026 年服务公告（2026-10-10 多源 DataAgent；2026-02-26 关闭数据湖集成入口）](https://help.aliyun.com/zh/maxcompute/2026-service-notices)
- [未央网：2026 主流数据中台产品深度横评（DataWorks Data Agent 细节）](https://www.weiyangx.com/472935.html)
- [51CTO：2026 企业级云数据库选型，瑶池数据库 6 大产品线行业适配指南，2026-09-04](https://blog.51cto.com/u_17622335/14909895)
- [AtomGit：阿里云瑶池数据库 KVCache 亮相 NVIDIA GTC 2026](https://gitcode.csdn.net/69c2448c0a2f6a37c599f782.html)
- [财新周刊：抢占办公室（千问办公整合史、企业上下文定位），2026-10-09](https://weekly.caixin.com/2026-10-09/102491720.html)
- [阿里云产品总览（官方）](https://www.aliyun.com/product)

---

*数据截至 2026-10-24。性能与规模数字除标注外均为阿里云官方发布口径，未经第三方验证。*
