# Agent Skill 调研：给大模型装"程序性知识"的开放格式

> **TL;DR**：Agent Skill 是把"怎么做一件事"的流程知识打包成文件夹（SKILL.md）的开放标准，靠三层渐进披露做到"装得多、花得少"。2025 年 10 月由 Anthropic 发明，两个月内微软、OpenAI 跟进，一年内 40+ 产品采纳，是近年采纳最快的互操作标准。行动建议：个人今天就可以把高频重复指令迁移成 Skill；团队把它当流程资产进 git 管理。

---

过去两年，让每个工程师反复做的一件苦差是：把同一段 400 字的指令（评审规范、输出格式、检查清单）一次又一次粘进新会话。我们在社区看到的极端案例：有用户个人目录下装了 49 个 Skill 仍无启动负担，而同样内容写进常驻提示词，5000 token 的税每轮都要交。

Agent Skill 给这条路画上了句号。本文核心判断：**它赢的原因不是技术复杂，而是简单到一个下午就能实现解析——简单性本身就是竞争力**。本文解释 Skill 是什么、解剖它的工作原理、给上手路径和写法要点，最后列出已知的坑。

理解 Skill 最好的类比来自 Anthropic 官方：**给新员工准备的入职手册**。手册分目录、章节、附录，新人先扫目录判断哪章有用，再读章节，需要时才翻附录。Skill 就是让模型这样读知识。

---

## 一、定义与边界

**一句话定义**：Agent Skill 是一种把"怎么做一件事"的流程知识打包成文件夹、让模型按需自动加载的开放格式，核心是 SKILL.md 文件。

最小实例（结构来自 Anthropic 官方 PDF skill）：

```
pdf-form-filler/
  SKILL.md       # 必需
  reference.md   # 可选：按需加载的参考
  forms.md       # 可选：特定场景才读
```

```markdown
---
name: pdf-form-filler
description: Fill out PDF forms. Use when the user provides
  a PDF form and asks to complete or sign fields.
---

# PDF Form Filling

1. Read `reference.md` for tool usage.
2. If the form has signature fields, read `forms.md` first.
3. Fill fields programmatically, never by hand-editing bytes.
```

一个文件夹 + 一个 `SKILL.md`（YAML frontmatter 必填 `name` 和 `description`，Markdown 正文写流程）+ 可选附属文件。**不需要注册、没有清单文件**，放进约定目录即生效——这就是上面那句话定义的全部内容。

**与相邻概念的辨析**——理解 Skill 的关键：

| 概念 | 解决什么 | 与 Skill 的关系 |
|---|---|---|
| Prompt | 一次性指令传递 | Skill 是"资产化的 prompt"：免粘贴、自动触发、可进 git |
| Tool / MCP | 连接外部工具和数据（连接层） | Skill 不连接任何东西，它教模型"用这些手该怎么做"（行为层） |
| Plugin | 打包分发一组能力 | Plugin 是分发单元（可含多个 Skill + MCP + Hooks），Skill 是内容单元 |
| RAG | 注入事实性知识（"是什么"） | Skill 注入程序性知识（"怎么做"） |
| Custom GPT / Gem | 平台锁定的定制化方案 | Skill 是开放文件格式，跨产品可移植——这是本质区别 |

边界：Skill 适合"流程稳定、可描述、重复发生"的任务。需要实时数据、强确定性计算或毫秒级延迟的场景，该用工具/MCP，不该写成 Skill。

---

## 二、何时用，何时不用

**适用**（至少满足一条就该写成 Skill）：

1. 同一套指令被粘贴过 3 次以上（写作规范、评审流程、输出格式）。
2. 流程知识只存在于老员工脑中或聊天记录里，人走即失。
3. 团队新人 onboarding 要重复讲解同一套操作规范。

**不适用**：

1. 需要实时数据或强确定性计算——Skill 交给模型判断力，这两种需求要确定性，用工具。
2. 一次性任务——写 Skill 的固定成本高于粘贴一次的收益。
3. 主模型能力弱——触发路由由模型完成，弱模型上 Skill 体验打折甚至误触发。

**Trade-off 要说透**：Skill 用"把路由交给模型"换来了免注册免配置，代价是触发质量与模型能力绑定，且触发错误本身有成本（模型猜错 Skill，执行跑偏）。这个代价在强模型上可忽略，在弱模型上是硬伤。

---

## 三、演进脉络

```
2022-23    系统提示词时代：定制能力 = 改系统提示，无版本、无共享
2023-11    OpenAI 推出 GPTs：定制化方案可分享，但锁死在平台内
2024-11    Anthropic 推出 MCP：标准化连接层（模型怎么够到工具）
2025-10-16 Agent Skills 首发（Claude Code + Claude 应用）
           动机公开：告诉模型"怎么按设计规范出图"
           比构建参数化刚性工具产出的 dashboard 更好——指令优于硬编码
2025-12-18 开放标准发布：agentskills.io + 开源参考 SDK
           公布后 48 小时：微软 VS Code、OpenAI（ChatGPT + Codex CLI）跟进
2026-03    32 家采纳：Gemini CLI、GitHub Copilot、Cursor、JetBrains Junie、
           Snowflake、Databricks、字节、Mistral
2026 年中  40+ 产品；第三方注册表 GuildSkills 收录 16 万+ Skill
2026-09    规范提案移交 Linux 基金会 Agentic AI Foundation（待表决）
```

**为什么胜出的是开放文件路线**：2023 年 GPTs 证明了"可分享的定制化"有需求，但平台锁定让换家归零，教训被反复引用。MCP 已经演示过同一剧本——连接层标准化后捐给 Linux 基金会，SDK 月下载 9700 万（2026-02）。Skill 只是把这套打法复制到行为层。备选路线（各家私有方案、API 化托管方案）在"竞争对手几周内敢不敢采纳"这个测试上全部不及格——没人敢把别人的私有 API 装进自己产品，人人都敢解析一个文件夹。

---

## 四、解剖：架构与原理

用一个实例贯穿：上面的 `pdf-form-filler`。模型处理"帮我把这份 PDF 表格填了"这句话时，发生的事分三层。

**三层渐进披露（Progressive Disclosure）**，整个设计的心脏：

- **第一层（元数据，常驻）**：会话启动时，模型只看到所有已装 Skill 的 `name` + `description`，每 Skill 约几十 token。50 个 Skill 只花 50 行短描述。
- **第二层（正文，命中即载）**：模型判断任务与描述匹配，才读取完整 `SKILL.md`。
- **第三层（附属文件，按需）**：正文引用的 `reference.md`、`forms.md`，需要时才读。只有填签名表才读 `forms.md`。

对比常驻指令：5000 token 写进系统提示，每轮固定付费；写成 Skill，平时 1 行，用到那一刻才付费。这个差值决定了"装得起 50 个 Skill，装不起 50 段常驻指令"。

**触发机制：description 即路由**。模型自己判断"该用哪个 Skill"，依据只有 description 这一行。因此 description 是全 Skill 杠杆最高的一行——同时决定"该触发时触发不触发"和"不该触发时误不误触发"。

**可执行性**：Skill 可附脚本，指令与确定性动作打包在同一文件夹。文本部分交给模型判断力，机械部分交给脚本——这是它比纯提示词资产多出来的一半能力。

**三个关键设计决策**：

1. **为什么是文件而不是 API**：文件可进 git、可 code review、可跨产品携带、可安全审计。这个"无聊"的选择是它被竞争对手快速采纳的直接原因。
2. **为什么是渐进披露而不是全量加载**：上下文膨胀（context rot）已被大量实测证实会损害模型表现。固定税（always-on）换成按需付费（on-demand）。
3. **为什么是模型路由而不是规则引擎**：语义匹配天然处理"同一个任务的千百种说法"，免注册免配置。代价见第二章 trade-off。

---

## 五、使用模式

Skill 按存放位置分三级，统一按"是什么 / 何时用 / 示例 / 注意"看：

### 5.1 个人级（`~/.claude/skills/`）

- **是什么**：存在用户主目录，跨所有项目可用。
- **何时用**：个人风格类资产——写作规范、代码风格、报告模板。
- **示例**：社区实测有用户单人装 49 个个人 Skill。
- **注意**：不随仓库分发，换机器要自行同步。

### 5.2 项目级（`.claude/skills/`）

- **是什么**：放在仓库里，随 git 共享。
- **何时用**：团队流程知识——评审规范、发布检查单、迁移脚本规范。
- **示例**：新人 clone 即获得团队全部流程知识，变更走 code review。
- **注意**：这是 ROI 最高的一级——组织过程知识第一次有了版本历史。

### 5.3 组织级（管理平台下发）

- **是什么**：企业平台统一下发和管理。
- **何时用**：合规要求强的行业（医药、金融）——合规流程、隐私规范各自成 Skill 且可组合。
- **示例**：Anthropic 首发即提供组织级管理能力。
- **注意**：需要配套的审核与版本治理，否则下发等于投毒。

---

## 六、生态格局

**标准与治理**：agentskills.io（规范 + 采纳清单），GitHub `agentskills/agentskills`，参考校验库 skills-ref；2026-09 提案移交 Linux 基金会 Agentic AI Foundation。

**采纳方**（截至 2026 年中，官方清单 40+ 产品）：

| 类别 | 代表 | 一句话定位 |
|---|---|---|
| 模型厂商 | Anthropic / OpenAI / Google / Mistral / 字节 | 首发者与跟进者，全部兼容同一格式 |
| IDE 与编码助手 | VS Code、GitHub Copilot、Cursor、JetBrains Junie | 编辑器率先支持（48 小时内） |
| Agent 运行时 | Claude Code、Codex CLI、Gemini CLI、Goose、OpenClaw | Skill 的主要消费端 |
| 数据与云平台 | Snowflake、Databricks | 把自家平台操作沉淀为 Skill |

**分发渠道**：skills.sh（`npx skills add <owner/repo>` 一键装）、Anthropic 官方 skills 仓库（PDF/DOCX/XLSX/PPTX 参考实现）、发布伙伴目录（Canva、Stripe、Notion、Zapier）、GuildSkills（16 万+）。

**本土关联**：Kimi K3 默认可读 SKILL.md；OpenClaw 完整实现 Skill 体系（本文写作流程本身由 Skill 驱动）；AWS agent-toolkit-for-aws 用 Skill 做数据库意图路由（见本博客《云数据库 MCP 全景调研》）。

---

## 七、对比评估

### Skill vs MCP：手与脑

| 维度 | MCP | Skill |
|---|---|---|
| 解决什么 | 连接：模型怎么够到工具和数据 | 行为：模型拿到手之后该怎么做 |
| 上下文成本 | 工具定义常驻（用不用都在） | 仅命中时加载 |
| 形态 | 服务/进程，有传输协议 | 文件夹，纯文件 |
| 确定性 | 高（API 语义） | 中（模型按指令执行） |

两者互补：MCP 给模型手，Skill 给模型流程。严肃 Agent 部署两个都用。

**决策建议**：如果你的问题是"模型够不到我的系统"→ 先 MCP；如果是"模型够得到但总是做错"→ 上 Skill；两者都痛 → MCP 打底、Skill 编排，这也是 AWS、阿里云的共同终局（见本博客云厂调研篇）。

### 自建 vs 采用第三方

- 组织内部流程知识必须自建：写一次永久受益。
- 通用能力（PDF/PPT 处理）用官方或高星仓库，但过供应链审查——见下章。

---

## 八、实践：上手与最佳实践

**最小上手路径**（三步，真实命令）：

```bash
mkdir -p ~/.claude/skills/my-first-skill
cat > ~/.claude/skills/my-first-skill/SKILL.md << 'EOF'
---
name: my-first-skill
description: 做什么 + 什么时候触发，写清楚这两件事
---
# 流程正文
1. 第一步做什么
2. 第二步做什么
EOF
```

放进目录即生效，无需重启、无需注册。

**最佳实践**（官方指南 + 社区实测）：

1. description 写"做什么 + 何时触发"，这是唯一决定 Skill 生死的一行。
2. 一个 Skill 只干一类事，避免大而全导致触发混乱。
3. 长知识拆进 reference 文件，正文保持短——正文是第二层，附属文件是第三层，别挤在一层。
4. 内置自我验证步骤（截图检查、来源核对）。
5. 修正写回：模型用错时，把纠正写进 SKILL.md 本体。一次性修复变成持久指令——这是 Skill 相对 prompt 最大的复利。

---

## 九、失败模式与坑

1. **Trigger confusion（触发混淆）**。怎么发现：该用的 Skill 没用、不该用的乱入。怎么绕：重写 description 加入明确触发词；把重叠的大 Skill 拆成单职责小 Skill。
2. **弱模型路由不准**。怎么发现：换模型后 Skill 命中率骤降。怎么绕：减少同域 Skill 数量、description 写得更显式；核心流程降级为强模型执行。
3. **第三方供应链风险**。2026 年初两篇学术研究：公开渠道超 1/4 Skill 至少含一个安全漏洞；确认恶意的 Skill 平均携带 4 种攻击向量（Skill 附带脚本 = 以你的权限在你机器执行任意代码）。怎么绕：只装官方或高星仓库；第三方 Skill 视同依赖库做代码审查；生产环境锁目录。
4. **目录约定不统一**。`.claude/skills`、`.codex/skills`、`.cursor/skills` 各家为政。怎么绕：正文跨平台无损，迁移时只重调运行配置；Cursor 2.4 起同时扫描三家目录。

---

## 十、结论与下一步

三行收束：Skill 把流程知识变成可版本化的文件资产；三层渐进披露解决了"装得多、花得少"；开放 + 简单让它一年内成为事实标准。

**给不同读者的下一步**：

- **开发者（你现在就能做）**：挑一个本周粘贴过 3 次以上的指令，15 分钟写成 Skill 装进个人目录；用一周，感受免粘贴 + 自动触发的差值。
- **团队负责人**：把评审规范、发布检查单迁进项目级 `.claude/skills/`，下一个新人的 onboarding 时间就是你的 ROI 证明。
- **产品经理（观望者）**：盯两个指标——你负责的产品用户有没有"反复教模型做同一件事"的行为（有就该开 Skill 入口）；Skill 触发成功率（这是新的功能质量维度）。

---

## 附录：FAQ

**Q1：Skill 和 MCP 会合并成一个标准吗？**
短期不会。两者解决不同层（连接 vs 行为），且都进了 Linux 基金会生态。长期看可能出现"打包格式"（plugin 层）把它们装在一起，Anthropic 的 plugin 机制已是雏形。

**Q2：一个 SKILL.md 写多长？**
没有硬上限，但正文建议控制在一次读取能消化的量级（社区经验 500 行以内），超出的知识拆进 reference 文件走第三层。长正文本身不违规，但浪费了渐进披露的设计。

**Q3：第三方 Skill 到底能不能装？**
能，但按依赖库标准审：看仓库来源、看脚本内容、最小权限运行。1/4 漏洞率是针对公开渠道整体，官方仓库和头部仓库的比率远低于此。

**Q4：和 Claude 的自定义指令、Projects 是什么关系？**
自定义指令 = 常驻指令（always-on 税）；Projects = 带知识库的工作区；Skill = 按需触发的流程资产。三者互补：常驻的放自定义指令，知识放 Projects，流程放 Skill。

**Q5：不用 Claude 能用 Skill 吗？**
能。格式是开放的，OpenAI、Google、微软的产品都支持，Kimi K3 默认可读 SKILL.md。这正是它相对 Custom GPTs 的本质优势。

---

## 参考资料

### 一手（官方文档 / 仓库）

1. [Anthropic Engineering: Equipping agents for the real world with Agent Skills（2025-10-16）](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
2. [Agent Skills 开放标准与规范](https://agentskills.io)；[GitHub](https://github.com/agentskills/agentskills)
3. [Anthropic 官方 Skills 仓库](https://github.com/anthropics/skills)

### 二手（解读 / 媒体 / 社区）

4. [VentureBeat: How Anthropic's 'Skills' make Claude faster, cheaper, and more consistent（2025-12-22）](https://venturebeat.com/ai/how-anthropics-skills-make-claude-faster-cheaper-and-more-consistent-for)
5. [AIToolsReview: Claude Skills Explained（2026-09-09，含 40+ 采纳清单）](https://aitoolsreview.co.uk/insights/claude-skills-explained)
6. [7minAI: How to Write a Claude Skill（2026-07-27，三层加载实测与个人/项目目录）](https://7minai.com/how-to-write-claude-skills/)
7. [MindStudio: Claude Agent Skills（2026-09-14，时间线与首发细节）](https://www.mindstudio.ai/blog/claude-agent-skills-anthropic-engineers)
8. [ishchuk.eu: Why Installing Third-Party AI Agent Skills Is Riskier Than You Think（2026-07-21，供应链安全研究转述）](https://ishchuk.eu/blog/why-installing-third-party-ai-agent-skills-is-riskier-than-you-think-in-2026)
9. [amdatalakehouse: Open Standards for Agentic Harnesses（2026-08-31，MCP/Skill 基金会进程）](https://amdatalakehouse.substack.com/p/open-standards-for-agentic-harnesses)
10. [Bosio Digital: The File Is the Easy Part（2026-09-28，规范字段与写作指引）](https://bosio.digital/articles/agent-skills)
11. [Articsledge: What Is Skill Engineering?（2026-07-29）](https://www.articsledge.com/post/skill-engineering)
