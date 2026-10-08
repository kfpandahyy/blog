# Agent Loop 调研：一段 while 循环里的全部工程学

> **TL;DR**：Agent loop 是让 LLM"推理→调工具→看结果→再推理"反复循环直到任务完成或预算耗尽的最小控制结构，2022 年 ReAct 论文把它形式化，此后四年所有主流 agent 都是它的变体。它简单到可以用一句话实现（"LLM 在 while 循环里调工具"），难在四件事：终止条件、错误复利（90% 单步准确率跑 10 步只剩约 35% 成功率）、上下文腐化、成本失控。2026 年"loop engineering"已成显学——设计的对象从提示词转向循环本身。选型一句话：能用单次调用解决的别上 loop，能用 workflow 解决的别上 agent，loop 的复杂度预算要花在 guardrail 上而不是模式花样上。
> **数据截至 2026-10。**

---

2026 年 6 月，OpenClaw 创始人 Steinberger 发了一句被浏览五百万次的话："别再给 coding agent 写提示词了。去设计会替你给 agent 写提示词的循环。"几乎同时，Claude Code 的创造者 Boris Cherny 说："我已经不给 Claude 写 prompt 了。我有一组在跑的循环，它们负责 prompt Claude。我的工作是让循环跑对。"（datasciencedojo 考据，2026-06-09）

两句话指向同一个判断：agent 时代的工程重心，正在从"写好一次提示词"迁移到"设计好一个循环"。而循环本身，这个被 ReAct 论文在 2022 年就写明白的东西，恰恰是大多数 agent 系统里最被低估、最容易写错、也最值得精读的一层。

本文的回答方式：先给 loop 的精确定义和它与 workflow 的分界线，再拆一次循环内部的五段结构与四个工程难点（终止、复利、腐化、成本），然后梳理从 ReAct 到 loop engineering 的演进谱系，最后给出实践清单与失败模式。

类比先行：**chatbot 是一问一答的自动售货机，agent loop 是学徒工——拿到任务先做一步，看一眼结果，想下一步，直到交活或被叫停。** 学徒和售货机的区别不在零件，在那条"看一眼再想"的回路上。

## 一、定义与边界

**Agent loop 是让 LLM 在控制循环中反复执行"感知上下文 → 推理 → 选择并调用工具 → 观察结果 → 更新状态"直到目标达成或停止条件触发的执行结构。** 最小实现确实是"while 循环里调 LLM + 工具"（Oracle 开发者博客定义，2026-03-16），权威路径以 ReAct（Yao et al., 2022）形式化：Thought / Action / Observation 三者交织，reasoning 与 acting 互相强化——推理指引行动，行动结果反哺推理。

边界必须划清（依据 Anthropic《Building Effective Agents》，2024-12，该文是此区分的事实标准）：

- **单次 LLM 调用**：一问一答，无环境交互，无自我检查。
- **Workflow**：LLM 与工具沿**预定义代码路径**编排——prompt chaining、routing、parallelization 都是 workflow，路径由程序员写死，LLM 只在节点内决策。
- **Agent（loop）**：LLM **动态指挥自己的执行过程与工具使用**，循环的路径由模型在运行中决定，程序只提供循环骨架与停止条件。
- **多 agent**：多个 loop 的组合结构（orchestrator-workers / handoff），是本博客《多智能体协作》的范畴，不是本文。

一句话分界：**workflow 的图是代码画的，agent 的图是模型画的，loop 是图能自己长出来的那条回路。**

## 二、何时用，何时不用

**该用 loop（agent）**：任务路径事先不可枚举、需要根据中间结果改道、错误恢复策略无法用规则穷举——例如"帮我把这个仓库的问题排查一下"，步骤数与分支都未知。

**不该用，按顺序降配**：

1. 单次调用 + RAG 能解决：多数问答场景。每加一层循环都在付延迟和成本的税。
2. 该用 workflow 解决：步骤可分解、可校验、路径固定——营销文案生成后翻译、按类别分流客服问题、并行跑多个独立子任务。workflow 给确定性与可测试性，agent 给灵活性，用灵活性换不可预测要算过账。
3. 该用 loop + 固定校验门解决：路径半开放但每步可程序化检查——在循环里插 gate，别把整个循环交给模型即兴。

**成本红线**（ranksquire 的提法值得记住）：agent 花 $10 算力解一个 $5 的任务，它不是资产，是玩具。loop 的每次迭代都要乘以"这步值多少钱"再跑。

## 三、演进脉络

| 时间 | 节点 | 贡献 |
|---|---|---|
| 2022-10 | ReAct（arXiv:2210.03629） | Thought/Action/Observation 交织循环形式化，agent loop 的出生证明 |
| 2023-03 | Reflexion（arXiv:2303.11366） | 失败后语言化自我反思写入情景记忆，下一轮表现提升——给 loop 加"复盘"段 |
| 2023-05 | Tree of Thoughts（arXiv:2305.10601）、Voyager（arXiv:2305.16291） | 前者把循环内的推理扩展为多路径搜索；后者在 Minecraft 里闭环"探索→建技能库→组合复用"，是 Hermes Agent 学习闭环的学术前身 |
| 2023-09 | LATS（arXiv:2309.17195） | 把推理树 + 行动 + 反思统一进 MCTS 式循环，单 agent 推理的集大成，代价是延迟与 token 翻倍 |
| 2024-12 | Anthropic《Building Effective Agents》 | 划出 workflow/agent 分界线，给出五种 workflow 模式——工程界开始区分"图是代码画的还是模型画的" |
| 2025 | OpenAI Agents SDK / Claude Agent SDK / LangGraph 等 | loop 成为 SDK 的一等公民；SDK 把循环骨架、工具协议（MCP）、会话与预算封装成标准件 |
| 2026 上半 | "loop engineering" 提法流行 | Steinberger "designing loops that prompt your agents"（2026-06-07，24 小时 500 万浏览）与 Cherny "my job is to write loops" 引发社区大讨论；datasciencedojo 梳理出十种 loop 变体（2026-06-09） |
| 2026 | Agentic RAG 成常规形态 | 检索从 pipeline 首步变为 loop 内可调工具 + critique gate 自我校验（2026-07 多篇架构文） |

演进主线一句话：**从"让模型在循环里想"（ReAct），到"让循环替人管模型"（loop engineering）**。视角转了 180 度——优化的对象从循环内的推理质量，变为循环本身的结构质量。

## 四、解剖：循环内部与四个工程难点

### 4.1 五段结构

综合 ReAct 与 OpenClaw 官方定义（后者为一手：intake → context assembly → model inference → tool execution → streaming replies → persistence），一次循环迭代可精确拆为五段：

1. **感知（Perceive）**：拼装上下文——用户目标、历史消息、工具结果、记忆注入、系统提示。注意 OpenClaw 把 context assembly 单列为一段：感知不是被动收消息，是主动的工程行为。
2. **推理（Reason）**：模型基于当前上下文决定下一步——继续、调哪个工具、给什么参数、还是交付答案。
3. **行动（Act）**：执行工具调用（含权限校验与沙箱）。
4. **观察（Observe）**：工具结果回写上下文，成为下一轮推理的输入。OpenClaw 把 streaming 与 persistence 也归入权威路径——每一步工具调用后即落盘，崩溃恢复不丢已完成步骤。
5. **判定（Decide）**：继续循环 / 交付 / 中止。停止条件是循环的"刹车"，没有刹车的循环只是烧钱器。

工程约束三条：**单会话串行**（OpenClaw、Hermes 均如此——并发消息排队进会话，防 loop 被打乱）；**迭代预算**（Hermes AIAgent 默认 max_iterations=90，预算耗尽强制终止）；**事件外发**（每次迭代的生命周期事件广播给多端 UI，可观测性是内生的）。

### 4.2 难点一：终止条件

循环没有天然终点。必备停止信号至少四种：模型主动声明完成（最不可信）；迭代数上限（硬顶）；token/成本预算（硬顶）；时间上限与外部中断（人按 stop）。生产系统四者都要有，且硬顶优先于模型自述——模型在困境里倾向于"再试一次"而不是认输。

### 4.3 难点二：错误复利

这是 loop 最硬的数学。Atlan 引 CMU 2025 基准：单步准确率 90% 的 agent，跑 10 步端到端成功率只剩约 35%（0.9^10 ≈ 0.35）；多步任务整体成功率仅 30-35%（2026-08-12）。推论：**loop 的长度和可靠性成指数trade-off，每多一步都在复利消耗信任**。工程对策只有三类——缩短路径（更好的规划与工具设计）、降低单步错误率（工具描述、校验门）、允许回滚（checkpoint/重试）。指望"模型变聪明"解决复利是逃避工程。

### 4.4 难点三：上下文腐化

循环迭代越多，历史越长，早期信息越容易被稀释或挤出窗口——context rot。对策已成标准件： compaction 摘要（压缩旧消息腾窗口，OpenClaw 压缩前先 memory flush）、子 agent 隔离（把长支线丢给独立上下文，主循环只回收结论）、checkpoint 回滚（LangGraph 式状态快照）。**Atlan 的判断值得记住：生产失败多发生在感知与观察段（上下文工程），不在推理段**——这与本博客《上下文与记忆工程》的结论互相印证。

### 4.5 难点四：成本与可观测

循环的计费单位是迭代。一次用户请求背后是 5-50 次模型调用（ranksquire 数据），成本可变且难预测。可观测三件套：逐步轨迹记录（trajectory）、每步 token 与延迟计量、事件流外发。没有轨迹的 loop 出问题是不可调试的。

## 五、组件模式拆解：推理结构谱系

统一微结构（一句话定位 / 循环特征 / 何时用）：

- **ReAct**（基础环）：推理与行动逐步交织；单路径、自适应最强、全程可见。通用默认选项，任何支持 function calling 的模型可跑。
- **Plan-and-Execute**：先出完整计划再逐步执行，中途可重规划；全局视野好、早期方向错代价大。步骤多且结构可预期的长任务。
- **Reflexion**：在 ReAct 外加"复盘段"——失败后生成反思写入记忆，影响后续轮次；允许跨 episode 改进。质量敏感、允许重试的任务（代码修复、写作迭代）。
- **Tree of Thoughts / LATS**：循环内展开多候选路径搜索（LATS 融合推理+行动+反思进蒙特卡洛树）；质量上限最高，延迟与成本数倍。难题求解、离线可并行的分析任务。
- **Agentic RAG loop**：检索降级为循环内工具，外加 critique gate（忠实度/覆盖度自检），不过门就改查询重检索。知识密集型问答，静态 RAG 的"检一次就答"不够用时。

选型判断（servicesground 对比，2026-09-25，结论与 Anthropic"从简到繁"一致）：**先用 ReAct 跑基线，发现哪类失败模式最痛再叠加对应结构**——返工多上 Reflexion，方向偏上 Plan-and-Execute，质量天花板不够再上 ToT。一上来叠全套是最常见的过度设计。

## 六、生态格局：框架怎么实现 loop

- **LangGraph**：把 loop 建成显式状态图，节点是计算、边是路由（含条件边与循环回边），checkpoint 一等公民。可控性最强，图由开发者定义——严格说是"workflow 引擎支持 agent 式动态边"。
- **OpenAI Agents SDK**：agent 作为循环实体，handoff 与 guardrail 内置，loop 对用户是隐式的——几行代码起一个会自己跑的 agent，黑盒程度也高。
- **Claude Agent SDK / coding agent harness**（本博客《Agent Harness 调研》）：loop 与文件系统、终端、审批流深度耦合，单会话串行 + 工具白名单。
- **OpenClaw / Hermes Agent**（一手）：loop 之上加渠道接入、记忆文件、定时触发——loop 从"被调用的函数"变成"长驻服务"，2026 年自托管 agent 产品的共同形态。
- **多 agent 编排**（manager / orchestrator-workers / handoff，Oracle 2026-03 梳理）：多个 loop 的拓扑组合。共识建议：单 loop 能解决的别上多 agent——每加一个 loop 就加一份复利错误与协调成本。

## 七、对比评估

| 维度 | 单 loop（ReAct 基线） | loop + 反思/搜索 | 多 agent 编排 | workflow |
|---|---|---|---|---|
| 路径决定者 | 模型逐步决定 | 模型（带回溯） | 模型 + 编排器 | 程序员 |
| 延迟 | 低 | 高（数倍） | 最高 | 最低 |
| 成本可预测性 | 中 | 低 | 低 | 高 |
| 可调试性 | 中（轨迹可读） | 低（分支多） | 低 | 高 |
| 适用 | 开放任务默认 | 质量敏感离线任务 | 真正需要专业分工 | 路径固定的生产任务 |

决策树：单次调用能解 → 别上 loop；路径可枚举 → workflow；路径开放但步骤少 → ReAct 单 loop；失败模式明确 → 对症下药叠结构；子任务可隔离且并发收益大于协调成本 → 才考虑多 agent。

## 八、实践：最小 loop 与检查清单

最小骨架（伪代码，约 30 行可落地）：

```python
state = [system_prompt, user_goal]
for i in range(MAX_ITER):            # 硬顶：迭代数
    resp = llm(state, tools=TOOLS)   # 推理段
    if resp.final_answer: break      # 模型自述完成
    result = run_with_guard(resp.tool_call)  # 行动段 + 权限校验
    state += [resp, observe(result)] # 观察段回写
    if budget_spent() > BUDGET: break        # 硬顶：成本
persist(trajectory)                  # 全程落盘，可回放
```

上线检查清单：四种停止信号齐了吗；每步轨迹落盘了吗；工具调用有权限边界吗；迭代预算与告警挂了吗； compaction 策略在窗口爆之前生效吗；失败任务能 checkpoint 重跑吗。

## 九、失败模式与坑

1. **无限循环型**：任务超纲，模型反复重试不认输。对策：迭代硬顶 + 连续重复动作检测（相同工具相同参数 N 次即中止）。
2. **工具幻觉型**：调用不存在的工具或编造参数（ranksquire 列为三大幻觉陷阱之一）。对策：工具 schema 强校验 + 调用前校验白名单。
3. **腐化漂移型**：跑了 30 步后忘了最初目标（goal drift）。对策：目标每轮重述进上下文 + 里程碑校验门。
4. **复利崩塌型**：单步 90% 撑不起十步任务却接了百步任务。对策：拆任务、插 gate、允许中途交付部分结果。
5. **成本爆炸型**：一次用户请求 50 次迭代烧掉整月预算。对策：成本预算硬顶 + 分级模型（简单步换小模型，Oracle 与 Anthropic 均推荐 routing）。
6. **黑盒失控型**：框架把 loop 包太深，出问题只能看最终答案。对策：选事件外发/轨迹一等的框架，或自己写薄封装——Anthropic 原话：先用 LLM API 直接实现，许多模式几行代码就够，框架的抽象层常常是调试障碍。

## 十、结论与下一步

判断：**agent loop 是 2022 年发明、2026 年才学会尊重的东西。** ReAct 早已给出循环的最小形式，但工程界花了四年才普遍接受三件事：循环的长度受错误复利约束、循环的可靠性感知观察段而非推理段、循环的设计对象（loop engineering）比循环内的提示词更值得投入。Steinberger 和 Cherny 的话之所以引爆，是因为它们宣告了分工的转移——人设计循环，循环指挥模型。

下一步盯三个信号：checkpoint/回滚与权限门是否成为 SDK 标配；错误复利的实测数据能否随模型代际改善（CMU 2025 的 30-35% 基线是否被刷新）；loop 级别的形式化验证（终止性证明、预算静态分析）会不会从论文走进工具。

给不同读者的一句话：

- **开发者**：先手写一个 30 行的 ReAct loop 再碰框架——没写过循环骨架的人调框架参数是盲调。
- **架构师**：把"单步准确率 → 步数 → 端到端成功率"的复利表贴进设计评审，任何超过五步的 loop 都必须给出缩短路径或插 gate 的方案。
- **产品经理**：loop 的经济学变了计费模型——从"每次调用"到"每次任务"，报价和 SLA 都要按任务口径重算。

## 附录：FAQ

1. **agent loop 和 ReAct 是一回事吗？** 不是。ReAct 是一种 loop 结构（推理行动交织）；loop 是更一般的控制结构，ReAct、Plan-and-Execute、Reflexion 都是它的变体。
2. **loop 和 agent 什么关系？** Anthropic 的定义：agent 是"LLM 动态指挥自身过程与工具使用"的系统——这个动态指挥能力正是由 loop 承载的。没有 loop 的"agent"只是 workflow。
3. **多大的任务值得上 loop？** 经验法则：预计步骤 ≤3 且路径已知 → workflow；步骤不可预知或需要观察改道 → loop。拿不准时先 workflow，失败模式指向"需要动态改道"再升级。
4. **loop 需要多 agent 吗？** 多数不需要。多 agent 解决的是上下文隔离与专业分工，代价是协调成本与复利错误翻倍。单 loop + 子任务工具调用能解的，别拆成多 loop。

## 参考资料

### 一手（论文 / 官方文档）

1. [ReAct: Synergizing Reasoning and Acting in Language Models（Yao et al., 2022）](https://arxiv.org/abs/2210.03629)
2. [Reflexion: Language Agents with Verbal Reinforcement Learning（Shinn et al., 2023）](https://arxiv.org/abs/2303.11366)
3. [Tree of Thoughts: Deliberate Problem Solving with Large Language Models（Yao et al., 2023）](https://arxiv.org/abs/2305.10601)
4. [Voyager: An Open-Ended Embodied Agent with Large Language Models（2023）](https://arxiv.org/abs/2305.16291)
5. [Language Agent Tree Search Unifies Reasoning, Acting, and Planning（LATS, 2023）](https://arxiv.org/abs/2309.17195)
6. [Anthropic: Building Effective Agents（workflow/agent 分界与五模式，2024-12）](https://www.anthropic.com/engineering/building-effective-agents)
7. [OpenClaw Agent Loop 概念文档（一手，六段权威路径）](https://docs.openclaw.ai/concepts/agent-loop)
8. [OpenAI Agents SDK 文档](https://openai.github.io/openai-agents-python/)

### 二手（分析 / 综述）

9. [Oracle Developers: What Is the AI Agent Loop（含 chatbot 分界问答，2026-03-16）](https://blogs.oracle.com/developers/what-is-the-ai-agent-loop-the-core-architecture-behind-autonomous-ai-systems)
10. [Atlan: The AI Agent Loop——架构与失败模式（CMU 错误复利数据，2026-08-12）](https://atlan.com/know/ai-agent/what-is-an-agent-loop/)
11. [DataScienceDojo: Agentic loops explained——从 ReAct 到 loop engineering（Steinberger/Cherny 言论考据，2026-06-09）](https://datasciencedojo.com/blog/agentic-loops-explained-from-react-to-loop-engineering-2026-guide/)
12. [servicesground: Agentic Reasoning Patterns 对比（ReAct/Reflexion/Plan-Execute/ToT，2026-09-25）](https://servicesground.com/blog/agentic-reasoning-patterns/)
13. [Agentic RAG 参考架构：循环内检索与 critique gate（2026-07-26）](https://iotdigitaltwinplm.com/agentic-rag-architecture-retrieval-agents-2026/)
14. [ranksquire: Agentic AI Systems 2026——循环、幻觉陷阱与成本红线](https://ranksquire.com/2026/01/22/agentic-ai-systems/)
