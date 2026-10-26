# blog

> 美观版在线阅读：<https://kfpandahyy.github.io/blog/>

AI Agent 领域技术博客，按九大类组织。文章按月份归档（如 `2026-10/`）。

学习路径：1 → 2 → 3 →（4 ∥ 5）→ 6 → 7 → 8。第 9 类为持续调研参考，不占学习顺序。

## 分类说明

| # | 分类 | 定位 | 覆盖内容 |
|---|------|------|----------|
| 1 | 模型与基座 | 大模型本身的能力与边界，agent 能力上限的决定者 | 模型系列（GPT / Claude / Llama / Kimi / Hermes）、架构（MoE vs Dense）、推理模型、benchmark 解读、能力边界（幻觉 / 上下文 / 指令遵循） |
| 2 | Agent 工程 | 把模型变成智能体的外层系统，整个知识体系的核心 | Agent loop、工具调用与 MCP 协议、记忆系统、规划与子 agent、自我验证、薄厚 harness 之争、线束债 |
| 3 | 框架与工具协议 | 业界怎么造 harness，看设计取舍而非只会调 API | LangGraph / LangChain 生态、Claude Agent SDK、OpenAI Agents SDK、MCP 详解、开源 harness 对比、编程 agent 产品拆解（Claude Code / Cursor / Codex） |
| 4 | 上下文与记忆工程 | agent 的工作记忆管理，长任务失败的常见死因 | 上下文窗口经济学（压缩 / 裁剪 / 摘要）、RAG vs 长上下文、长期记忆三种实现、context rot 治理 |
| 5 | 多智能体协作 | 何时需要多个 agent，防过早进入多 agent 的弯路 | 协作模式（orchestrator-worker / 辩论式 / 角色分工）、Cognition《Don't Build Multi-Agents》争议、通信与任务分配、单 agent 极限 |
| 6 | 评测与可观测 | 知道 agent 好不好，坏了怎么查 | 评测集（SWE-bench / AgentBench / OSWorld）、tracing 工具（Langfuse / LangSmith）、失败模式分类、eval 设计原则 |
| 7 | 安全与治理 | 生产环境生死线，harness 是唯一强制边界 | prompt injection 与防御、权限模型（沙箱 / 审批 / 最小授权）、OWASP LLM / Agent Top 10、多 agent 信任链、审计合规 |
| 8 | 产品化与商业 | 从 demo 到生意，前七类决定行不行，这类决定值不值钱 | 延迟 / 成本 / 可靠性三角、产品形态（编程 / 客服 / 数据 / 个人助理）、商业模式（seat / 量 / 结果）、案例复盘、团队组织 |
| 9 | 业界情况调研 | 技术与产品之外的全景情报，判断方向比打磨细节更重要 | 玩家图谱（模型厂 / 云厂 AWS·阿里云 / 创业 / 开源）、融资估值、产品追踪对比、市场采用数据、人才信号、监管动向 |

## 总览

按分类组织，编号见上表；类内最新在前。第 9 类按云厂商细分。

### 1 模型与基座

1. [《知识蒸馏调研：让大模型把"判断力"传给小模型》](./2026-10/2026-10-14-knowledge-distillation.md)
2. [《Hermes 是什么：一个开源大模型系列的简史》](./2026-10/2026-10-07-hermes-intro.md)

### 2 Agent 工程

1. [《Agent Loop 调研：一段 while 循环里的全部工程学》](./2026-10/2026-10-08-agent-loop.md)
2. [《Agent Harness 是什么：给大模型装上"身体"的那层软件》](./2026-10/2026-10-07-agent-harness.md)

### 3 框架与工具协议

1. [《MCP 调研：一个 18 个月大的协议，为什么把有状态握手整个删掉了》](./2026-10/2026-10-18-mcp.md)
2. [《OpenClaw 调研：从一小时原型到基金会，多渠道 Agent 网关成了自托管基础设施》](./2026-10/2026-10-08-openclaw.md)
3. [《Hermes Agent 调研：会自己写技能的 agent，"越用越聪明"是不是噱头》](./2026-10/2026-10-08-hermes-agent.md)
4. [《Agent Skill 调研：给大模型装"程序性知识"的开放格式》](./2026-10/2026-10-08-agent-skill.md)

### 4 上下文与记忆工程

1. [《WeKnora：腾讯把"RAG 查询、Agent 干活、Wiki 沉淀"缝进同一个知识库》](./2026-10/2026-10-09-weknora.md)
2. [《LLM Wiki：Karpathy 的"知识编译器"范式，以及它与 RAG、本体、Agent 记忆的关系》](./2026-10/2026-10-09-llm-wiki.md)
3. [《DBAIOps 代码仓考古：cfe9d30，论文背后的 7.6 万行工程底座》](./2026-10/2026-10-09-dbaiops-repo.md)
4. [《DBAIOps 精读：知识图谱能力如何被量化评测》](./2026-10/2026-10-09-dbaiops-paper.md)
5. [《本体与知识图谱调研：RAG 答不了"谁依赖谁"，答它靠图》](./2026-10/2026-10-08-ontology-knowledge-graph.md)

### 9 业界情况调研

#### 阿里云

1. [《Agent Context 调研：阿里云把 Agent 的记忆做成了数据库生意》](./2026-10/2026-10-22-aliyun-agent-context.html)
2. [《阿里云 AI + Data 产品体系全景：一张图看懂 60+ 产品及关系》](./2026-10/2026-10-20-alibaba-cloud-ai-data-system.md)
3. [《瑶池数据库 Agent 调研：阿里云的"工单+大模型"路线，与 DBAIOps 的"图谱+推理"路线对位》](./2026-10/2026-10-09-apsaradb-agent.md)

#### AWS

1. [《AWS Data 与 AI 产品体系调研：三层 Agent 栈、一分为二的 SageMaker 与一个 Iceberg 赌注》](./2026-10/2026-10-21-aws-data-ai.md)
2. [《AWS 数据库 Skill 深度解剖：把解决方案架构师的选型决策写成代码》](./2026-10/2026-10-20-aws-database-skill.md)
3. [《AWS 数据库 MCP 全景:工具、Skill 与托管网关的三层版图》](./2026-10/2026-10-18-aws-databases-mcp.md)
4. [《AWS 数据库 MCP 全景调研》](./2026-10/2026-10-08-aws-database-mcp-skills.md)

#### 火山引擎

1. [《火山引擎 Data 与 AI 产品体系调研：四层架构、五线模型与一条数据产品线》](./2026-10/2026-10-21-volcengine-data-ai.md)
2. [《Seedance 调研：字节视频生成模型的版本版图、能力拆解与场景边界》](./2026-10/2026-10-21-seedance.md)
3. [《扣子（Coze）调研：字节 Agent 平台的版本演进、生态版图与锁定风险》](./2026-10/2026-10-21-coze.md)

#### 百度

1. [《百度秒哒深度调研：无代码应用生成平台的头号玩家，靠多智能体把"想法"变成生意》](./2026-10/2026-10-20-baidu-miaoda.md)

#### 其他

1. [《OceanBase AI 功能全景：存量能力盘点、2026 年新增清单与未来趋势判断》](./2026-10/2026-10-26-oceanbase-ai-features.md)
2. [《TiDB AI 功能全景：存量能力盘点、2026 年新增清单与未来趋势判断》](./2026-10/2026-10-26-tidb-ai-features.md)
3. [《CPU、GPU、TPU、NPU：四类处理器的架构、定位与选型》](./2026-10/2026-10-23-cpu-gpu-tpu-npu.md)
4. [《FDE（前线部署工程师）调研：从 Palantir 的战场起源到 AI 时代的战略岗位》](./2026-10/2026-10-21-fde.md)
5. [《波士顿矩阵（BCG Matrix）调研：结构、原理、场景与五十年的批评》](./2026-10/2026-10-21-bcg-matrix.md)
6. [《羲和 HaloDB 调研：功能全景、优势拆解与主打场景》](./2026-10/2026-10-20-halodb-database.md)
7. [《Databricks 调研：Lakehouse 巨头正把自己重做为企业 AI Agent 的基础设施》](./2026-10/2026-10-18-databricks.md)
8. [《uv 调研：一个 Rust 二进制收编了 Python 工具链，OpenAI 随后买下了它》](./2026-10/2026-10-17-astral-uv.md)
9. [《Supabase 技术调研：Postgres 优先的 Agentic Backend 平台》](./2026-10/2026-10-13-supabase-tech-research.md)

---

写作模板见 [templates/](./templates/)：技术点调研（八章）与业界调研（六章）两套。
