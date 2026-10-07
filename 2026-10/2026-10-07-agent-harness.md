# Agent Harness 是什么：给大模型装上"身体"的那层软件

> **TL;DR**：Agent Harness 是包裹在大模型外围的软件层——循环、工具、上下文、记忆、沙箱、护栏，业内公式是 Agent = Model + Harness。2026 年初"线束工程"一词定型后，它从隐性工程实践变成显学；核心争论是薄 harness（信模型）还是厚 harness（控流程）。行动建议：评估任何 Agent 产品先看它的 harness——审批流、沙箱、上下文策略、模型可插拔度，这四项比 benchmark 分数更能预测生产表现。

---

2026 年 2 月，Mitchell Hashimoto 发表文章给 "harness engineering"（线束工程）定型，OpenAI 几天后发布 Codex harness 实践报告，LangChain 创始人 Harrison Chase 随后给出定义 **Agent = Model + Harness**。一个词在两周内聚齐三方声音，说明大家造了三年的东西，终于到了需要名字的时候。

本文核心判断：**模型能力每上一个台阶，harness 不会消失，只会换位**——从"替模型补短板"转向"管模型的权限和记忆"。本文讲清 harness 是什么、拆它的六大部件、梳理薄厚之争的实质，最后给评估方法和已知的坑。

类比是这篇文章的地基：**模型是大脑，harness 是身体**。大脑决定想得多好，身体决定能碰到什么、记得多少、闯了祸谁兜底。

---

## 一、定义与边界

**一句话定义**（维基百科，2026）：**Agent Harness（也叫 agent scaffolding，智能体脚手架）是包裹在大语言模型外围的软件基础设施，负责工具调用、记忆、状态持久化、执行环境和反馈循环——模型本身不推理之外的一切。**

最小实例——一个能跑的最简 harness 伪代码，核心就三层：

```python
while not done and steps < MAX_STEPS:        # ① 循环与停止条件
    resp = model(context)                     # 模型决策
    action = parse(resp)                      # ② 解析工具调用意图
    if action.type == "tool":
        result = sandbox_exec(action)         # ③ 沙箱内执行，结果回填
        context.append(result)
    elif action.type == "final":
        done = True                           # 停止
```

真实世界的 harness 是这三层乘以百倍复杂度：Claude Code 的 harness 部分约有 **51.2 万行代码**（源码泄露事件中披露的数）——全世界模型做得最好的公司，照样在"模型外的那层"上重金投入。

**is / is-not 辨析**：

| 概念 | 与 harness 的关系 |
|---|---|
| MCP | 是 harness 用来接工具的标准协议之一，属于 harness 的一个部件 |
| Agent 框架（LangGraph / Claude Agent SDK） | 是建造 harness 的半成品材料，不等于 harness 本身 |
| Agent 产品（Claude Code / Cursor） | = harness + 模型 + 产品化界面 |
| Prompt / Skill | 装进 harness 的知识与指令资产（见本博客《Agent Skill 调研》） |

**词源**（避免和亲戚搞混）：测试领域的 test harness（测试夹具）是祖爷爷；LLM 评测圈的 lm-evaluation-harness 是父辈；agent harness 是同一家族最新的用法——每个场景都指"让被执行对象能干活、且能被观察的那套架子"。

---

## 二、何时用，何时不用

**适用**——任务满足"多步、需工具、状态跨轮"任一特征，就该有 harness：

1. 多步任务且步骤无法预先写死（模型需要边看边决策）。
2. 需要调用外部工具或执行代码。
3. 任务跨会话（需要记忆与状态恢复）。

**不适用**：

1. 单次调用就能完成的任务——套 agent 循环纯属浪费延迟和 token，一个 prompt 解决。
2. 步骤完全确定的流程——用普通代码/workflow 编排，确定性吊打模型自主决策。
3. 对正确性零容忍且无法加人工审批的场景——任何 harness 都救不了概率性错误，这类需求不该用 Agent。

**Trade-off 说透**：harness 每厚一分，可控性增一分，"模型自主红利"减一分。薄厚没有绝对答案，只有和你场景匹配的厚度——这正是第四章之争的实质。

---

## 三、演进脉络

```
2023       UK AISI 把智能体描述为"模型 + 脚手架"；METR 提出
           "scaffolding + model = agent"——概念已存在，名字是散的
2024-11    Anthropic 发布 MCP：工具接入标准化，harness 的工具层有了通用协议
2024-12    Anthropic《Building Effective Agents》：工作流与 Agent 的架构区分
           成为行业共识，harness 的"编排层"有了理论骨架
2026-02    Mitchell Hashimoto 定型 "harness engineering"；OpenAI 发布 Codex
           harness 实践报告；Harrison Chase 给出 Agent = Model + Harness——
           词聚齐三方，实践升级为显学
2026 春    薄厚之争公开化（Anthropic vs LangGraph 两派）；"harness debt"
           （线束债）一词出现——第一代为绕模型局限写的补丁开始到期
2026 年    共识成型：harness 是唯一强制安全边界（安全侧），同时是
           性能天花板来源（债务侧）——同一层软件，两种属性
```

路径选择：同期存在"把一切塞进模型"的路线（更长的上下文、更强的指令遵循，让 harness 趋于零）。实验结果（见下章 Opus 4.5 案例）否定了这条路线——不是永久否定，而是"当前模型能力下不成立"。harness 的厚度本质是"模型能力缺口"的函数。

---

## 四、解剖：一个 Harness 的六大部件

以"模型要在一个陌生仓库里完成一个功能需求"为贯穿实例，看六大部件各自干什么、真实世界里长什么样。

### 4.1 Agent 循环（Loop）

驱动"模型决策 → 执行动作 → 观察结果 → 再决策"的主循环，含停止条件、最大步数、错误恢复。这是 harness 的心脏。真实形态：Claude Code 的主循环、ReAct 模式的工程化版本。设计决策：**为什么循环要保持极简**——循环里每加一段硬编码分支，就是在对未来的自己关死一扇门（见 harness debt）。

### 4.2 工具调度（Tool Dispatch）

把模型的"我想调用某工具"变成真实动作，结果回填。MCP 正在成为工具接入的标准协议——注意 MCP 规范自己写明：工具"代表任意代码执行"。设计决策：**为什么工具要走标准协议而不是私有适配**——harness 的可插拔性（模型可换、工具可换）来自接口标准化，这正是 MCP 一年被 40+ 产品采纳的原因（见本博客《云数据库 MCP 全景调研》）。

### 4.3 上下文管理（Context Management）

决定每一步往上下文窗口里放什么：历史、检索结果、工具返回、压缩摘要。Anthropic 称之为 context engineering，视为 agent 工程的核心子学科。设计决策：**为什么上下文策略比模型选择更影响长任务成败**——见下面的实验。

**关键实验**：Anthropic 发现，不给 harness 的 Opus 4.5，无法仅凭一个高层 prompt 跨多个上下文窗口完成一个生产级 Web 应用。两种典型死法：一是"一次性梭哈"，上下文在中途耗尽，代码写了一半，下个会话面对烂摊子浪费 token 猜进度；二是**假成功**——后期会话看到部分成果就宣布完成，从不验证任何东西。这个实验是"光有大脑不够"的最硬证据。

### 4.4 记忆与状态（Memory & State）

跨会话的文件、数据库、进度日志，让"上次干到一半"可恢复。LangChain 的说法很直白：向 harness 要求"插上记忆"，就像向汽车要求"插上驾驶"——记忆本来就该长在 harness 里。

### 4.5 执行沙箱（Sandbox）

代码在哪里跑、文件能访问哪里、网络权限多大。harness 决定模型动作发生在什么隔离级别。

### 4.6 护栏（Guardrails）

权限范围、审批门槛、人工确认、监控审计。**这是唯一一层可以强制执行的约束**——模型拒绝恶意指令只是一种"偏好"，harness 拒绝执行才是一种"控制"。安全研究（Adversa）扫描了 10 多个 AI IDE 和编程助手，发现 30 多个漏洞，每一个都命中 harness 层，没有一个需要模型本身有缺陷。prompt injection 攻破模型是时间问题，业界至今没有可靠的通用模型侧防御——所以结论很冷：**harness 是 agent 安全的真正边界**。

---

## 五、模式拆解：薄、厚与中间派

统一微结构（是什么 / 何时用 / 示例 / 注意）：

### 5.1 薄 Harness 派（Anthropic 为代表）

- **是什么**：循环保持极简，把尽可能多的决策交还给模型。
- **何时用**：任务边界模糊、需要模型灵活发挥；模型快速迭代的团队。
- **示例**：Claude Code 的基础形态——主循环 + 工具 + 文件系统，复杂行为靠模型在循环里涌现。
- **注意**："未来兼容"是它的核心论据——模型升级后 agent 自然变强，无需重构逻辑。

### 5.2 厚 Harness 派（LangGraph 为代表）

- **是什么**：显式状态图、确定性控制流编排复杂工作流。
- **何时用**：对可靠性、可审计、可调试要求高的生产系统。
- **示例**：多步骤审批链、带回滚的发布流程、强合规场景。
- **注意**：过厚的代价见下章"未来兼容测试"。

### 5.3 中间派（OpenAI Agents SDK、CrewAI 等）

- **是什么**：关键环节显式化（状态检查点、审批），其余交给模型。
- **何时用**：大多数生产场景的实际答案。
- **示例**：带 human-in-the-loop 门槛的自主执行。
- **注意**："关键环节"选哪几处，是这类 harness 设计的全部功力。

---

## 六、生态格局

**产品侧**（你叫得出名字的编程 agent 严格说都是 harness）：

| 产品 | harness 特征 |
|---|---|
| Claude Code | 薄循环代表，51.2 万行 harness 代码 |
| OpenAI Codex | 云端沙箱 + 审批流 |
| Cursor / aider / OpenCode | IDE 集成型 harness |
| OpenClaw（本文运行环境） | 网关 + 工具注册 + 多渠道 + 记忆文件 + 定时任务 |

**框架侧**（造 harness 的半成品）：

| 框架 | 定位 |
|---|---|
| Claude Agent SDK | 自我定位"通用 agent harness" |
| LangGraph / Deep Agents | 厚编排 + 内置规划工具、虚拟文件系统、子 agent 生成 |

---

## 七、对比评估

### 薄 vs 厚

| 维度 | 薄 harness | 厚 harness |
|---|---|---|
| 可控性 | 低（模型说了算） | 高（状态图说了算） |
| 可调试性 | 难（行为涌现） | 易（路径显式） |
| 模型升级红利 | 直接继承 | 被逻辑层吃掉一部分 |
| 适合场景 | 探索、创作、快速迭代 | 合规、生产、强审计 |

**核心检验标准——未来兼容测试**：把当前模型换成更强的下一代，agent 表现是否**不增加 harness 复杂度**就变好？薄 harness 应通过此测试；过厚的通不过——你为旧模型写的每一段绕路逻辑，都在阻止新模型发挥。

### 决策建议

if 场景容错低、要审计 → 厚，且把审批流当一等公民；
if 场景探索性强、模型迭代快 → 薄，把循环外的一切当可疑复杂度；
if 拿不准 → 中间派，只把"出错代价最高的环节"显式化。

---

## 八、实践：评估与构建

**评估一个 harness，看四眼**（按优先级）：

1. 审批流：哪些动作需要人确认？粒度细不细？
2. 沙箱：代码执行在什么隔离级别？文件网络权限多大？
3. 上下文策略：满了怎么办——截断、压缩、还是摘要？有没有防假成功的验证步骤？
4. 模型可插拔度：换模型要动几处代码？（答案大于"一处配置"就要警惕锁定）

**构建顺序建议**：先写最小循环（本文第一章的伪代码能跑），再接工具，再加记忆，最后加护栏。顺序反过来做的人，大多把护栏做成了绊马索。

---

## 九、失败模式与坑

1. **Harness debt（线束债）**。Anthropic 平台团队 Lance Martin（2026-04）："harness 编码的是'模型自己做不到什么'的假设，这些假设会随模型变强而腐化。"怎么发现：agent 还能跑、eval 还能过，但新模型的能力被闲置。怎么绕：定期做未来兼容测试；给每段 harness 补丁写"失效条件"注释。
2. **一次性梭哈与假成功**（Opus 4.5 实验的两种死法）。怎么发现：长任务成功率随上下文长度衰减；agent 报告完成但未验证。怎么绕：强制验证步骤、分段检查点、进度落盘。
3. **护栏做成形式**。审批弹窗点麻了就全点同意。怎么绕：审批粒度按"出错代价"分级，只对不可逆动作设卡。
4. **复杂度失控**。51.2 万行是头部产品的重量，不是目标。怎么绕：把"循环外的每段逻辑"都视为需要论证的复杂度。

---

## 十、结论与下一步

三行收束：harness = 模型外的一切，是把无状态预测器变成有状态执行者的那层软件；薄厚之争的实质是"信模型还是控流程"，答案由场景的出错代价决定；harness 同时是唯一强制安全边界和性能天花板来源——建它时两手都要硬。

**给不同读者的下一步**：

- **开发者**：用 50 行 Python 写一个最小 agent 循环（读文件 + 执行命令 + 回填结果），跑通后再去看 Claude Code 的复杂度——你会知道自己站在哪一层。
- **团队负责人**：把"未来兼容测试"加进季度工程复盘：换用更强模型，你们的 agent 是否白拿提升？没白拿，说明 harness debt 在累积。
- **产品经理**：评估 Agent 供应商时，用第八章的"四眼"代替 demo 体验——demo 展示的是模型能力，四眼检验的是交付能力。

---

## 附录：FAQ

**Q1：harness 和 scaffolding 是一个东西吗？**
是。scaffolding（脚手架）是 2023-2024 年的叫法（METR、UK AISI 在用），harness（线束）是 2026 年定型后的主流叫法。测试领域的 test harness 是更早的同名前辈。

**Q2：MCP 是 harness 吗？**
不是。MCP 是 harness 里"工具调度"这一部件的接入协议。harness 还包含循环、上下文、记忆、沙箱、护栏，MCP 一概不管。

**Q3：模型足够强之后，harness 会消失吗？**
证据指向"换位而非消失"。Opus 4.5 实验表明当前最强模型无 harness 仍过不了长任务；同时护栏作为强制边界的属性与模型能力无关——注入攻击不靠模型缺陷就能命中 harness 层。会消失的是"补短板型" harness，"管权限型" harness 长存。

**Q4：Claude Code 的 51.2 万行都是 harness 代码吗？**
泄露事件中看到的 51.2 万行是其工程实现的总量级，harness 相关逻辑占主体。这个数字的启示是量级：头部产品的"模型外软件"以十万行计，轻视这一层是新手最常见的误判。

---

## 参考资料

### 一手（官方博客 / 定义来源）

1. Wikipedia: Agent harness — https://en.wikipedia.org/wiki/Agent_harness
2. Anthropic: Building Effective Agents（2024-12）— https://www.anthropic.com/engineering/building-effective-agents
3. Anthropic: Effective harnesses for long-running agents — https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents
4. LangChain: Your harness, your memory（2026-04）— https://www.langchain.com/blog/your-harness-your-memory
5. Mitchell Hashimoto: Harness engineering（2026-02）— https://mitchellh.com/writing

### 二手（综述 / 解读 / 安全研究）

6. Databricks: What is an AI Agent Harness? — https://www.databricks.com/blog/ai-harness
7. Firecrawl: What Is an Agent Harness? — https://www.firecrawl.dev/blog/what-is-an-agent-harness
8. Adversa: What is an agent harness?（含 30+ 漏洞扫描）— https://adversa.ai/blog/what-is-an-agent-harness/
9. Gentic News: The Great Agent Harness Debate — https://gentic.news/article/agent-harness-debate-anthropic-vs
10. AI Heroes: Harness Debt — https://www.ai-heroes.co/en-us/blog/ai-agent-harness-debt-2026
11. DataNorth: Harness Engineering — The Complete Guide — https://datanorth.ai/blog/harness-engineering-the-complete-guide-to-ai-agent-scaffolding
12. Vector Culture: A Brief History of Agents — https://vectorculture.substack.com/p/a-brief-history-of-agents
