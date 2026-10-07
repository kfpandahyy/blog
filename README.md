# blog

AI Agent 领域技术博客，按八大类组织。

学习路径：模型与基座 → Agent 工程 → 框架与协议 →（上下文工程 ∥ 多智能体）→ 评测与可观测 → 安全与治理 → 产品化与商业。

## 总览

| 类型 | 文章 | 日期 |
|------|------|------|
| 1 模型与基座 | [Hermes 是什么：一个开源大模型系列的简史](./2026-10/2026-10-07-hermes-intro.md) | 2026-10-07 |
| 2 Agent 工程 | [Agent Harness 是什么：给大模型装上"身体"的那层软件](./2026-10/2026-10-07-agent-harness.md) | 2026-10-07 |

---

## 1. 模型与基座（LLM）

大模型本身的能力与边界。Agent 的一切能力上限由模型决定，不懂模型，后面的讨论都是空中楼阁。

覆盖：主流模型系列（GPT / Claude / Llama / Kimi / Hermes）、架构（MoE vs Dense）、推理模型的兴起、benchmark 怎么读（哪些有参考价值、哪些被刷烂）、能力边界（幻觉、上下文长度、指令遵循）。

- 2026-10-07 · [Hermes 是什么：一个开源大模型系列的简史](./2026-10/2026-10-07-hermes-intro.md)

## 2. Agent 工程（Harness 与运行时）

把模型变成智能体的外层系统。Agent = Model + Harness，这一类是整个知识体系的核心。

覆盖：Agent loop（ReAct 模式、reason-act-observe）、工具调用与 MCP 协议、记忆系统（短期/长期）、规划与子 agent、自我验证与反馈循环、薄厚 harness 之争、线束债（harness debt）。

- 2026-10-07 · [Agent Harness 是什么：给大模型装上"身体"的那层软件](./2026-10/2026-10-07-agent-harness.md)

## 3. 框架与工具协议

业界怎么造 harness。看清各框架的设计取舍，而不是只会调 API。

覆盖：LangGraph / LangChain 生态、Claude Agent SDK、OpenAI Agents SDK、MCP 协议详解、开源 harness 对比（OpenClaw / OpenCode / Deep Agents）、编程 agent 产品拆解（Claude Code / Cursor / Codex 的 harness 各有什么特点）。

## 4. 上下文与记忆工程

agent 的"工作记忆"管理。长任务做不好，八成死在这里。

覆盖：上下文窗口经济学（压缩 / 裁剪 / 摘要的时机）、RAG 与长上下文之争、长期记忆的三种实现（文件系统 / 向量库 / 知识图谱）、context rot 的成因与治理、context engineering 作为独立学科。

## 5. 多智能体协作

何时需要多个 agent，何时一个就够。多数人过早进入多 agent，是常见弯路，这一类放后面学。

覆盖：协作模式（orchestrator-worker / 辩论式 / 角色分工）、反面教材 Cognition《Don't Build Multi-Agents》及其争议、agent 间通信与任务分配、单 agent 的能力极限在哪。

## 6. 评测与可观测

怎么知道 agent 好不好，以及坏了的时候怎么查。

覆盖：评测集（SWE-bench / AgentBench / OSWorld）、tracing 与日志工具（Langfuse / LangSmith）、失败模式分类学（context rot / tool overload / 过早宣布成功 / 脆弱的工具接线）、eval 设计原则（别用 demo 骗自己）。

## 7. 安全与治理

生产环境的生死线。模型拒绝恶意指令只是概率性行为，harness 才是强制边界。

覆盖：prompt injection 与已知防御手段、权限模型（沙箱 / 审批 / 最小授权）、OWASP LLM / Agent Top 10、多 agent 环境下的信任链、审计与合规要求。

## 8. 产品化与商业

从 demo 到生意。前七类决定技术行不行，这一类决定值不值钱。

覆盖：延迟 / 成本 / 可靠性的不可能三角、agent 产品形态（编程 / 客服 / 数据分析 / 个人助理）、商业模式（按 seat / 按量 / 按结果）、案例复盘（Devin 们兑现了什么、没兑现什么）、agent 团队的组织形态。
## 9. 业界情况调研

技术与产品之外的全景情报。判断方向比打磨细节更需要它，属于持续跟踪类而非学习路径类。

覆盖：主要玩家图谱——模型厂商（OpenAI / Anthropic / Google / 微软）、云厂商（AWS / 阿里云 / Azure / GCP，含各自的 agent 平台如 Bedrock AgentCore、百炼等）、创业公司与开源格局；融资动态与估值变化；新产品发布追踪与横向对比；市场数据（企业采用率、付费意愿、落地案例）；人才流动与招聘信号；开源社区趋势；监管与政策动向。

---

*目录结构：1-8 为学习路径（模型与基座 → Agent 工程 → 框架与协议 →（上下文工程 ∥ 多智能体）→ 评测与可观测 → 安全与治理 → 产品化与商业），第 9 类为持续调研参考。*
