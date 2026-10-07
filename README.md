# blog

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

| 类型 | 文章 | 日期 |
|------|------|------|
| 1 模型与基座 | [Hermes 是什么：一个开源大模型系列的简史](./2026-10/2026-10-07-hermes-intro.md) | 2026-10-07 |
| 2 Agent 工程 | [Agent Harness 是什么：给大模型装上"身体"的那层软件](./2026-10/2026-10-07-agent-harness.md) | 2026-10-07 |
| 9 业界情况调研 | [云数据库 MCP 全景调研：AWS 与阿里云](./2026-10/2026-10-08-aws-database-mcp-skills.md) | 2026-10-08 |
| 3 框架与工具协议 | [Agent Skill 调研：给大模型装"程序性知识"的开放格式](./2026-10/2026-10-08-agent-skill.md) | 2026-10-08 |

---

写作模板见 [templates/](./templates/)：技术点调研（八章）与业界调研（六章）两套。
