# 阿里云 AI + 数据全家桶：一页看懂 60+ 产品及它们的关系

> 一句话定位：阿里云是国内唯一"从芯片到应用入口"全链路自研的云厂商，AI 与数据两条产品线在 2026 年已经合并成一张图。本文只做两件事：列出全系产品，说清它们怎么咬合。不做深度评测，只看结构和关系。

## 一、总图：七层结构

```
L5  应用入口    千问App · 千问办公 · 钉钉AI · 夸克 · QwenNote · 手机/车机(Qwen Intelligence)
L4  数据产品    瑶池数据库(PolarDB/RDS/PolarDB-X/AnalyticDB/Lindorm/Tair) · MaxCompute · Hologres · DataWorks · 向量检索全家桶
L3  平台与运行时 千问AI平台(原百炼) · PAI · AgentCore · Agent Sandbox · 无影AgentBay · 魔搭ModelScope
L2  模型        Qwen 家族(Max/Next/Coder/VL/Omni/Audio/Image/Wan视频/百聆)
L1  智算基建    灵骏集群 · 磐久超节点 · HPN网络 · CPFS · OSS · 飞天+CIPU
L0  芯片        真武V900/M890(AI) · 倚天(CPU) · ICN Switch · 磐脉(网卡) · 镇岳(存储主控)
```

层间关系一句话：**L0→L2 解决"智能怎么生产"，L3 解决"智能怎么编排"，L4 解决"智能吃什么数据"，L5 解决"谁在用"**。

## 二、分层清单（每层只看三件事：是什么、给谁用、一句话）

### L0 芯片（平头哥）

| 产品 | 是什么 |
|---|---|
| 真武 V900 | 新一代训推一体 AI 芯片，性能为 M890 三倍，2027 Q1 量产 |
| 倚天 710 | 自研服务器 CPU |
| ICN Switch | 自研互联芯片，千卡全带宽互联 |
| 磐脉 / 镇岳 | 智能网卡 / SSD 存储主控 |

定位：对标 NVIDIA 全栈（GPU + NVLink + BlueField）。意义不在单品性能，而在"芯片—集群—模型"可以协同设计（Qwen3.8-Max 已演示自主做芯片物理实现优化，面积减 42%）。

### L1 智算基础设施

| 产品 | 是什么 |
|---|---|
| 灵骏 | 智算集群品牌，单一集群可扩至 50 万卡（2026 云栖口径） |
| 磐久超节点 | 集成 V900 + ICN Switch + 磐脉 + 镇岳的整机柜 |
| HPN 8.0 | 阿里云数据中心网络架构 |
| CPFS | 高性能并行文件存储（训练用），2026 云栖发布新一代 |
| OSS | 对象存储，兼作向量数据底座（Vector Bucket） |
| 飞天 + CIPU | 自研云操作系统 + 云基础设施处理器 |

### L2 模型（Qwen / 通义家族）

- **旗舰**：Qwen3-Max（万亿参数）、Qwen3.8-Max（已实现 33 轮零人工自迭代训练）；Qwen4 在训
- **效率**：Qwen3-Next（80B 总参 / 3B 激活）
- **垂直**：Coder（代码）、VL（视觉 Agent）、Omni（全模态实时交互）、Audio-3.1 家族（ASR/TTS/Realtime/LiveTranslate 同传）、Qwen-Image-3.1（图像）、Wan 万相（视频，2026-11 发新版）、通义百聆（语音开源，下载 5.6 亿+）
- **分发**：魔搭 ModelScope，17 万+ 模型、2500 万开发者、9200+ MCP Server——Qwen 触达开发者的主渠道

### L3 平台与 Agent 运行时

| 产品 | 是什么 | 给谁用 |
|---|---|---|
| 千问AI平台（原百炼） | 模型服务 + Agent Studio（开发/运行/上下文/工具/评测/安全六环节）+ Agent API | 应用开发者，客户数年增 6 倍 |
| PAI | 机器学习平台（训练/微调/部署），深度学习工程师主阵地 | 算法工程师 |
| AgentCore | Agent 构建与治理平台（2026 云栖新品） | 企业级 Agent 生产 |
| Agent Sandbox | Agent 隔离运行环境（2026 云栖新品） | 同上 |
| 无影 AgentBay | 云端桌面/浏览器，Agent 的操作界面 | Agent 执行层 |
| ADK / ADP | Agent 开发框架（代码）/ 低代码平台（20 万开发者、80 万+ Agent） | 两类开发者 |

### L4 数据产品（全家桶的主体）

**瑶池数据库六线**（统一品牌，统一 DAS 自治运维）：

| 产品 | 一句话 |
|---|---|
| PolarDB / PolarDB-X | 云原生关系库 / 分布式事务库；AI 化最激进：Agentic Database、Agent Memory（兼容 Mem0）、LakeBase（Agent Workspace） |
| RDS | 通用托管 MySQL/PG，入门首选 |
| AnalyticDB | 云数据仓库，向量引擎 + NL2SQL |
| Lindorm | 多模融合（宽表/时序/搜索），IoT 万亿行 |
| Tair | 内存库（Redis 兼容）；KVCache 下沉给 LLM 推理破显存墙 |
| DAS | 跨库自治诊断调优 |

**大数据与治理**：MaxCompute（离线数仓）+ Hologres（实时数仓）湖仓双引擎；DataWorks（开发治理，2026 全面 Agent 化：Data Agent / 运维 Agent / SQL 事前检查）；周边 EMR、Flink、SLS、Quick BI、OpenSearch。

**向量检索**：DashVector、OSS Vector Bucket、Tablestore（Agent 记忆/知识库）、AnalyticDB/Lindorm/Tair 库内向量、PolarSearch。策略很明确：**不做独立向量库单品，让每个存储都长向量能力**——数据在哪，向量就在哪。

### L5 应用与入口

| 产品 | 一句话 |
|---|---|
| 千问 App | C 端助手 |
| 千问办公 | 企业 Agent 平台 + **企业上下文（Enterprise Context）**：把企业数据压缩结构化，供数字员工实时调用；配 QwenNote 随身硬件 |
| 钉钉 AI | 办公入口，数字员工与人类协同的 IM |
| 夸克 | 搜索/学习入口 |
| Qwen Intelligence | 手机/车机端侧 Agent 全栈方案（荣耀 Magic9 首发） |

## 三、产品之间的关系（本文重点）

三层流向，看清就理解了整家公司：

**1. 训练流（自下而上，生产智能）**
业务数据（L4：MaxCompute 数仓、OSS 语料）→ 灵骏集群（L1）→ Qwen（L2）。芯片（L0）决定这条链的成本上限。魔搭把训好的模型分发给社区，反哺 Qwen 的生态位。

**2. 服务流（自上而下，消费智能）**
用户（L5）→ 千问AI平台 / AgentCore（L3）→ 运行时调用 Qwen（L2）+ 从 L4 取上下文与记忆。**L3 是中枢**：它左手调模型、右手查数据，Agent 的工程问题（编排、工具、状态）都在这一层解决。

**3. 回流（闭环，数据越用越厚）**
Agent 和应用的运行数据写回 L4：行为日志进 Lindorm/SLS，业务结果进 PolarDB，治理后沉淀为企业上下文再喂给千问办公。**L4 因此从"成本中心"变成"智能的原料厂"**——这是阿里 Data 产品线在 AI 时代的真实位置。

四个关键耦合点：

- **Tair × 推理**：数据库技术反哺 AI。KVCache 从显存下沉到分布式内存，直接降推理成本
- **PolarDB × Agent**：AI 反哺数据库。Branch 穿梭=Agent 试错隔离，Agent Memory=上下文持久化，MCP/CLI 原生交互=Agent 直接驱动数据库
- **DataWorks × 千问办公**：治理成果（血缘、口径、元数据）是企业上下文的前置输入
- **魔搭 × 千问AI平台**：开源获客、平台变现的分工闭环

## 四、按场景选型速查

| 我要做 | 直接看这套组合 |
|---|---|
| 企业级 Agent | 千问AI平台（Agent Studio）+ PolarDB Agentic Database/Tablestore（状态与记忆） |
| 自然语言取数 | DataWorks Data Agent + MaxCompute 多源 DataAgent |
| RAG / 语义检索 | 数据在哪就用哪个库的向量能力；独立场景 DashVector |
| 模型微调/训练 | PAI + 灵骏；开源模型从魔搭拉 |
| 端侧 Agent（手机/车） | Qwen Intelligence 整套方案 |

## 五、边界与风险

- **生态绑定**：DataWorks 等仅公有云可用，数据治理成果难迁出；全家桶是优势也是锁
- **命名漂移快**：百炼→千问AI平台、通义→Qwen/千问双品牌、瑶池→六条产品线总称，文档与实物常对不上
- **组织变动**：应用层产品重组频繁（千问办公即整合产物），长期 roadmap 可信度需打折
- **口径提醒**：集群规模、性能倍数均为 2026 云栖官方口径，未经第三方验证

## 参考资料

- [2026 云栖大会首日发布合集（真武V900 / AgentCore / CPFS / 企业上下文），DoNews，2026-09-22](https://www.donews.com/news/detail/1/6720181.html)
- [财新：阿里发布智能体电脑等多款产品，2026-09-23](https://www.caixin.com/2026-09-23/102488138.html)
- [阿里云产品总览（官方）](https://www.aliyun.com/product)
- [InfoQ：2026 年的云栖，阿里给 AI 行业递了一把新尺子，2026-09-23](https://www.infoq.cn/article/zSPQbRwmS0AGpeTAhu9r)

---

*数据截至 2026-10-24。规模与性能数字除标注外均为阿里云官方发布口径。*
