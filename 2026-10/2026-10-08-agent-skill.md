# Agent Skill 调研：给大模型装"程序性知识"的开放格式

> 一句话结论：Skill 是把"怎么做一件事"的流程知识打包成文件夹（SKILL.md）的开放标准，靠三层渐进披露做到"装得多、花得少"，2025 年 10 月由 Anthropic 发明，两个月后成为全行业采纳最快的互操作标准。

---

## 一、问题背景

Agent 时代之前，让模型稳定完成一件重复性专业任务，只有一条路：prompt 工程。这条路有四个结构性痛点：

1. **重复消耗**：同一段 400 字的指令（品牌规范、评审流程、输出格式），每个新会话都要重新粘贴。
2. **知识私有**：个人调好的 prompt 存在自己的备忘录里，团队无法共享，人走即失。
3. **上下文税**：把指令写进系统提示或 CLAUDE.md，每轮对话都计费——无论这次任务是否相关。
4. **隐式知识无法资产化**：发布检查单、故障分诊流程、代码评审规范，这些组织里最值钱的过程性知识，只存在于老员工脑中和一次性聊天记录里。

Skill 出现的前置条件在 2025 年成熟：Agent Harness（见本博客《Agent Harness 是什么》）给模型配上了文件系统读写能力，模型可以像读文档一样"读指令文件"；同时模型的指令遵循能力足以按描述自行判断该不该加载某段流程。两个条件缺一，Skill 这种"让模型自己管理自己的知识加载"的设计都不成立。

---

## 二、定义与边界

**定义**：Skill 是一个文件夹，内含一个必需的 `SKILL.md` 文件（YAML frontmatter 声明元数据，Markdown 正文写流程指令），可选附带脚本、参考文档、模板等资源。Agent 启动时只读每个 Skill 的元数据（几十 token），判断任务匹配后才加载正文，需要时才读附属文件。

**与相邻概念的辨析**——这是理解 Skill 的关键：

| 概念 | 解决的问题 | 与 Skill 的关系 |
|---|---|---|
| Prompt（提示词） | 一次性指令传递 | Skill 是"资产化的 prompt"：不用粘贴，自动触发，可进 git |
| Tool / MCP | 连接外部工具和数据（连接层） | Skill 不连接任何东西，它教模型"用这些手该怎么做"（行为层） |
| Plugin（插件） | 打包分发一组能力 | Plugin 是分发单元（可含多个 Skill + MCP + Hooks），Skill 是内容单元 |
| RAG | 注入事实性知识（"是什么"） | Skill 注入程序性知识（"怎么做"） |
| Custom GPT / Gem | 平台锁定的定制化方案 | Skill 是开放文件格式，跨产品可移植，这是本质区别 |

边界：Skill 适合"流程稳定、可描述、重复发生"的任务。对于需要实时数据、强确定性计算、或毫秒级延迟的场景，该用工具/MCP，不该写成 Skill。

---

## 三、演进脉络

Skill 不是凭空出现，它是一次明确的路径选择的结果：

1. **2022-2023：系统提示词时代**。定制能力 = 改系统提示，无版本管理、无共享机制。
2. **2023-11：OpenAI 推出 GPTs**。定制化方案可分享了，但锁在 OpenAI 平台内，换平台归零——这个教训后来被反复引用。
3. **2024-11：Anthropic 推出 MCP（Model Context Protocol）**。标准化了连接层：模型怎么接外部工具和数据。一年后捐给 Linux 基金会，2026 年 2 月 SDK 月下载量达 9700 万。
4. **2025-10-16：Anthropic 推出 Agent Skills**。首发在 Claude Code 和 Claude 应用（Pro/Max/Team/Enterprise），内置 Excel、PPT、Word、PDF 生成能力（2025 年 9 月上线的文档功能底层已经是 Skill，只是未公开）。发布伙伴含 Box、Canva、Notion、Rakuten。团队公开的动机：告诉模型"怎么查数据、按什么设计规范出图"，比构建参数化的刚性工具产出的 dashboard 更好——**指令优于硬编码**。
5. **2025-12-18：开放标准**。规范 + 参考 SDK 发布在 agentskills.io，仓库开源。同一剧本第二次上演：开放管道，而不是死守私有标准。
6. **2026：全行业采纳**。公布后 48 小时内，微软在 VS Code 支持、OpenAI 在 ChatGPT 和 Codex CLI 支持（Simon Willison 发现 ChatGPT 代码解释器环境里有 /home/oai/skills 目录）。2026 年 3 月 32 家采纳，年中约 40-46 家：Google Gemini CLI、GitHub Copilot、Cursor（2.4 版同时扫描 .claude/skills、.codex/skills、.cursor/skills）、JetBrains Junie、Amp、Goose、Snowflake、Databricks、字节、Mistral。第三方注册表 GuildSkills 收录超 16.7 万个 Skill。2026 年 9 月，规范提案移交给 Linux 基金会下的 Agentic AI Foundation 治理。

直接竞争对手在几周内采纳同一格式，这在工业史上罕见。原因在下一章。

---

## 四、架构与原理（核心章节）

### 4.1 格式规范

```
my-skill/
  SKILL.md          # 唯一必需文件
  reference.md      # 可选：按需加载的参考文件
  scripts/          # 可选：可执行脚本
  assets/           # 可选：模板等资源
```

`SKILL.md` 的 frontmatter 只有两个必填字段：`name` 和 `description`；可选字段如 `allowed-tools`（限制该 Skill 可用工具）、`disable-model-invocation`（禁止模型自主触发）等。正文是 Markdown 流程指令。目录约定：个人级 `~/.claude/skills/`，项目级 `.claude/skills/`（随 git 共享）。**不需要注册、没有清单文件**——把文件夹放进去就生效。

### 4.2 三层渐进披露（Progressive Disclosure）

这是整个设计的心脏，三层加载模型：

- **第一层（元数据，常驻）**：会话启动时，模型只看到所有 Skill 的 name + description。50 个 Skill 只花 50 行短描述。
- **第二层（正文，命中即载）**：模型判断任务与描述匹配时，读取完整 SKILL.md。
- **第三层（附属文件，按需）**：正文引用的参考文件、脚本，需要时才读。

对比 CLAUDE.md：一个 5000 token 的指令块写进 CLAUDE.md，每轮固定花 5000 token；写成 Skill，平时只花 1 行，用到那一刻才付费。这个差值决定了"可以装很多 Skill，但装不起很多常驻指令"。

### 4.3 触发机制：描述即路由

由模型自己判断"该用哪个 Skill"，依据只有 description 这一行。因此 description 是整个 Skill 杠杆最高的一行——它同时决定"该触发时会不会触发"和"不该触发时会不会误触发"。社区称之为 trigger confusion（触发混淆）：描述含糊或互相重叠时，模型猜错或漏用。官方建议描述里写清"做什么 + 什么时候用"。

### 4.4 可执行性

Skill 可以附带脚本，指令与执行能力打包在同一文件夹。这使 Skill 不只是"读给模型听的文档"，而是"文档 + 确定性动作的混合体"——文本部分交给模型的判断力，机械部分交给脚本。

### 4.5 三个关键设计决策

1. **为什么是文件而不是 API**：文件可进 git、可 code review、可跨产品携带、可被安全审计。这个"无聊"的选择是它能被竞争对手快速采纳的直接原因——一个下午就能实现解析。
2. **为什么是渐进披露而不是全量加载**：上下文膨胀（context rot）已被大量实测证实会 degrades 模型表现。固定税（always-on）换成按需付费（on-demand），装 50 个 Skill 的成本从"不可能"变成"几乎免费"。
3. **为什么是模型路由而不是规则引擎**：免注册、免配置，语义匹配天然处理"同一个任务的千百种说法"。代价是把路由质量绑定在模型能力上，弱模型上 Skill 体验打折。

---

## 五、生态格局

**标准与治理**：agentskills.io（规范 + 文档 + 采纳清单），GitHub 仓库 agentskills/agentskills，参考校验库 skills-ref；2026-09 提案移交 Linux 基金会 Agentic AI Foundation。

**采纳方分类**（截至 2026 年中，约 40+ 产品）：

- **模型厂商**：Anthropic（首发）、OpenAI（ChatGPT + Codex CLI）、Google（Gemini CLI / Gemini 应用）、Mistral、字节
- **IDE 与编码助手**：VS Code、GitHub Copilot、Cursor、JetBrains Junie、Windsurf、Amp、Roo Code、Trae、Kiro
- **Agent 运行时与框架**：Claude Code、Codex CLI、Goose（Block）、OpenCode、Hermes Agent、OpenClaw
- **数据与云平台**：Snowflake（Cortex Code）、Databricks（Genie Code）

**分发渠道**：skills.sh（`npx skills add <owner/repo>` 一键安装）、Anthropic 官方 skills 仓库（PDF/DOCX/XLSX/PPTX 参考实现）、发布伙伴目录（Canva、Stripe、Notion、Zapier 等）、GuildSkills 等第三方注册表（16 万+ Skill）。

**本土关联**：Kimi K3 默认可读 SKILL.md；用户当前使用的 OpenClaw 运行时就完整实现了 Skill 体系（扩展插件自带 skills 目录，本博客写作流程本身已由 Skill 驱动）；AWS 的 agent-toolkit-for-aws 用 Skill 做数据库意图路由（见本博客《云数据库 MCP 全景调研》）。

**安全研究（2026 年初，两篇学术论文）**：公开渠道超四分之一的 Skill 至少含一个安全漏洞；确认恶意的 Skill 平均携带 4 种攻击向量。分发生态跑在了安全基础设施前面。

---

## 六、对比评估

### Skill vs MCP：手与脑

| 维度 | MCP | Skill |
|---|---|---|
| 解决什么 | 连接：模型怎么够到工具和数据 | 行为：模型拿到手之后该怎么做 |
| 上下文成本 | 工具定义常驻（用不用都在） | 仅命中时加载（渐进披露） |
| 形态 | 服务/进程，有传输协议 | 文件夹，纯文件 |
| 确定性 | 高（API 语义） | 中（模型按指令执行） |

两者互补而非竞争：MCP 给模型手，Skill 给模型流程。严肃 Agent 部署两个都用——这与本博客 AWS 调研篇的结论一致（aws-database Skill 负责路由，MCP 负责执行）。

### 自建 vs 采用第三方 Skill

- 自建：组织内部流程知识（评审规范、发布检查单）必须自建，写一次永久受益，且描述质量决定一切。
- 第三方：通用能力（PDF 处理、PPT 生成）用官方或高星仓库的，但要过供应链审查——四分之一漏洞率不是小数。最小化原则：Skill 附带的脚本等价于在你机器上以你的权限执行任意代码。

### 各厂商实现差异

核心文件通用，周边不通用：目录位置（.claude/、.codex/、.cursor/）、调用规则、权限设置各自为政。跨平台迁移时 Skill 正文无损，运行配置要重调。

---

## 七、场景与实践

**个人**：把反复粘贴的指令迁移为 Skill（写作规范、代码风格、报告模板）；实测用户单人装 49 个个人 Skill 而无启动负担。修正写回——模型用错时，把纠正写进 SKILL.md 本体，一次性修复变成持久指令。

**团队**：项目级 `.claude/skills/` 随仓库分发，新人 clone 即获得团队全部流程知识。变更走 code review，知识资产第一次有了版本历史。

**企业**：组织级目录 + 行业化 Skill 库（医药合规、临床试验分析、患者数据隐私各自成 Skill 并可组合）；发布检查单、故障分诊、迁移脚本规范是 ROI 最高的三类。

**写法要点**（来自官方指南与社区实测）：

1. description 写"做什么 + 何时触发"，这是唯一决定 Skill 生死的一行
2. 一个 Skill 只干一类事，避免大而全导致触发混乱
3. 长知识拆进 reference 文件，正文保持短
4. 在 Skill 里内置自我验证步骤（截图检查、来源核对）
5. 附带脚本的 Skill 视同可执行代码做安全审查

**已知的坑**：描述重叠引发误触发；弱模型路由不准；第三方 Skill 供应链风险；不同产品的目录约定不一致导致迁移要手动调整。

---

## 八、趋势与启示

1. **采纳速度本身是信号**。48 小时两大竞品跟进、一年内 40+ 产品——不是因为 Skill 技术复杂（恰恰相反，简单到可以一下午实现），而是因为"共享 Skill 库对所有厂商的客户都有利"。简单性就是它的竞争力。
2. **行业默认架构正在定型：MCP 管连接，Skill 管行为**。两层分离已成为 Agent 系统设计的公理。做 Agent 产品，架构图里少了哪层都要被问为什么。
3. **"Skill 工程"正在职业化**。从 prompt 工程（调一句话）到 skill 工程（维护一个可版本化的流程资产），配套出现注册表、排行榜、校验库、可视化编辑器（如 Frontmatter）。预测：明年会出现 Skill 质量评估与触发率指标的岗位需求。
4. **治理走向中立**。跟随 MCP 的脚步进 Linux 基金会，标准战争阶段结束，进入生态竞争阶段——胜负手在分发渠道与质量基础设施（谁解决那四分之一的漏洞率，谁赢得企业市场）。
5. **对产品工作的启示**：评估自家产品要不要暴露 Skill 能力，看一个判据——你的用户是否有"反复教模型做同一件事"的行为。有，就该给 Skill 入口；另外 Skill 库是把"客户成功经验"产品化的天然载体，这比功能迭代便宜得多。

---

## 参考资料

1. Anthropic Engineering: Equipping agents for the real world with Agent Skills（2025-10-16 发布）— https://www.anthropic.com/news/agent-skills （工程深读：https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills）
2. Agent Skills 开放标准与规范 — https://agentskills.io ；GitHub 仓库 — https://github.com/agentskills/agentskills
3. Anthropic 官方 Skills 仓库（内置 Skill 参考实现）— https://github.com/anthropics/skills
4. Simon Willison 关于 OpenAI 采纳 Skill 格式的发现（2025-12）— 经 inference.sh 转述：https://inference.sh/blog/skills/agent-skills-overview
5. VentureBeat: How Anthropic's 'Skills' make Claude faster, cheaper, and more consistent（2025-12-22）— https://venturebeat.com/ai/how-anthropics-skills-make-claude-faster-cheaper-and-more-consistent-for
6. AIToolsReview: Claude Skills Explained（2026-09-09，含采纳清单）— https://aitoolsreview.co.uk/insights/claude-skills-explained
7. 7minAI: How to Write a Claude Skill（2026-07-27，三层加载模型与个人/项目目录实测）— https://7minai.com/how-to-write-claude-skills/
8. MindStudio: Claude Agent Skills — How Anthropic Builds Reusable AI Workflows（2026-09-14）— https://www.mindstudio.ai/blog/claude-agent-skills-anthropic-engineers
9. ishchuk.eu: Why Installing Third-Party AI Agent Skills Is Riskier Than You Think（2026-07-21，供应链安全研究转述）— https://ishchuk.eu/blog/why-installing-third-party-ai-agent-skills-is-riskier-than-you-think-in-2026
10. amdatalakehouse: Open Standards for Agentic Harnesses（2026-08-31）— https://amdatalakehouse.substack.com/p/open-standards-for-agentic-harnesses
11. Bosio Digital: The File Is the Easy Part（2026-09-28，规范字段与写作指引）— https://bosio.digital/articles/agent-skills
12. Articsledge: What Is Skill Engineering?（2026-07-29）— https://www.articsledge.com/post/skill-engineering
