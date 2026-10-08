# Hermes Agent 调研：会自己写技能的 agent，"越用越聪明"是不是噱头

> **TL;DR**：Hermes Agent 是 Nous Research 2026 年 2 月开源的自进化 agent（Python / MIT），差异化不在渠道也不在模型，而在一个内置学习闭环——任务后自动创建 Skill、使用中自改进、跨会话检索自己的历史。它把"经验 → 能力"的回路做成了产品，并用独立的 GEPA 离线进化管道（ICLR 2026 Oral）补上运行时自学的两个致命弱点（过度乐观、覆盖人工技能），进化结果以 PR 形式提交、人类合并。选型一句话：要一个越用越懂你、且肯把进化闸门交给你的单 agent，选它；要多渠道运维成熟度或组织级多 agent，先看 OpenClaw。
> **数据截至 2026-10。** 项目周更节奏，版本与星数以 GitHub 为准。

---

先澄清一个混淆：本文的 Hermes Agent 不是本博客《Hermes 是什么》里那个 Hermes 模型系列。两者同出 Nous Research——前者是模型基座，后者是跑在任何模型上的 agent 框架，共享品牌而已。调研中见到的"Hermes 跑 Hermes"（用 Hermes 模型驱动 Hermes Agent）正是这种套娃的产物。

2026 年 2 月，agent 框架已经多到让人麻木：接 LLM、挂工具、编排流程，大同小异。Hermes Agent 的出场姿势不一样，README 就一句话："the only agent with a built-in learning loop"——唯一内置学习闭环的 agent。它声称自己会在干完活后复盘、把经验写成 Skill、越用越强。这个 claim 大到像噱头。本文的路线：先拆这个闭环到底由哪些机制组成（创建、改进、记忆、检索），再看它用什么约束防止闭环失控（GEPA 离线进化 + 人类 PR 闸门），最后与 OpenClaw 正面交锋给出选型判断。

类比先行：**如果说普通 agent 是每次都从说明书起步的临时工，Hermes Agent 试图当的是下班后会写工作笔记、把笔记整理成 SOP、定期回头修订 SOP 的老员工。** 问题只有一个：它写的笔记靠不靠谱。

## 一、定义与边界

**Hermes Agent 是 Nous Research 开源的自进化个人 agent：内置学习闭环，能从任务经验中自主创建并持续改进 Skill，持久化知识并跨会话检索自身历史，同时通过统一网关接入 Telegram / Discord / Slack / WhatsApp / Signal / CLI 等渠道。**（定义依据：官方 README 与文档，2026-10）

边界四条：

- **它是具体 agent，不是框架库**。官方原话定位 "a real agent" 而非 "yet another framework"——你安装的是成品，不是积木。
- **它不绑模型**。Nous Portal / OpenRouter（200+ 模型）/ OpenAI / Anthropic / 自有端点，`hermes model` 一键切换，无锁定。
- **它的进化有边界**。运行时闭环只创建与微调 Skill；结构性进化（工具描述、系统提示词段、工具代码）走独立的 self-evolution 仓库，且所有产出以 PR 提交、人类合并——agent 没有直接 commit 权。
- **它与 Hermes 模型系列的关系**是同品牌延伸：模型提供"大脑材质"，agent 提供"成长机制"，两者可独立选用。

## 二、何时用，何时不用

**适合用**：

- 你长期用同一批工作流（数据分析、代码审查、运维巡检），希望 agent 记住"我们家的做法"而不是每次重新教。
- 你只有一台 $5 VPS 或想跑 serverless（Modal / Daytona 支持休眠唤醒，闲置近零成本）。
- 你乐意审 PR：愿意把技能进化的最终闸门握在手里，换来自进化的收益。
- 你已经在用 OpenClaw，想迁走——`hermes claw migrate` 一键导入 SOUL.md、记忆、Skills、API key、渠道配置。

**不适合用**：

- 你要的是组织级多 agent 权限体系（delegate / 审计 / 身份分离）：它的多 agent 是任务协作视角（看板 + 角色），不是治理视角。
- 你不审 PR 也不想做安全基线：运行时自动创建 Skill 意味着 agent 在持续改写自己的程序性知识，无人值守的失控成本真实存在。
- 你要飞书 / 钉钉 / 企业微信等国内渠道开箱即用：渠道适配约 18 个，国内覆盖靠社区桥（如 HermesClaw 微信桥），成熟度弱于竞品。
- 你要"今天装明天忘"：自进化有累积效应，用一周和用半年的差距才是它的价值曲线，浅尝者无感。

**核心 trade-off**：用进化和审 PR 的注意力，换长期个性化收益。不审 PR 的人不该开自动进化。

## 三、演进脉络

| 时间 | 事件 |
|---|---|
| 2025-07-22 | 仓库创建（掘金深度解析文考据） |
| 2026-02-25 | 正式发布，主打 "built-in learning loop"（知乎专栏，2026-02-25） |
| 2026-02 底 | 首月破 2.2 万星（极客公园，2026-04-11） |
| 2026-03-17 | v0.3.0：实时 token 流式、一等公民插件架构、provider 系统重构 |
| 2026-04-08 | v0.8.0 单日新增 6400+ 星，总星破 4.7 万，多日霸榜开源榜第一（极客公园） |
| 2026-05 中 | 约 15 万星，GitHub 全球排名 47（掘金，基于 v0.13.0，2026-05-14） |
| 2026-05 | v0.13.0：Kanban 多智能体看板（心跳/僵尸检测/幻觉门控）；GEPA 论文获 ICLR 2026 Oral |
| 2026-07-31 | v0.16.0（lynxflow 对比文）；与 OpenClaw 被社区并列为两大自托管 agent |
| 2026-10 | 持续周更；社区衍生 HermesClaw（同微信账号共跑 Hermes 与 OpenClaw）、computer-use-linux 等 |

两个值得记的非技术信号：一是星数增速曲线（首月 2.2 万 → 两月 4.7 万 → 半年 15 万级）说明"自进化"叙事精准击中了 2026 年 agent 用户的疲劳点——不是再要一个框架，而是要一个能攒经验的伙伴；二是 `hermes claw migrate` 的存在说明它的战略假想敌非常明确——直接抢 OpenClaw 的用户资产（记忆、技能、密钥、渠道配置），这比功能对标的杀伤力大得多。

## 四、解剖：学习闭环与它的约束

学习闭环是全文核心，拆成四层机制 + 一套约束。

### 4.1 运行时闭环（agent 活着的时候）

- **触发**：复杂任务完成后（约 5 个以上工具调用）、发现非平凡工作流时、用户纠正或错误恢复后——agent 自动从轨迹中提取模式，创建可复用 Skill（agentskills.io 开放格式，与本博客《Agent Skill 调研》所述格式兼容，可被任何遵循该标准的宿主读取）。
- **改进**：Skill 在后续使用中持续被修订；`curator.py` 负责技能生命周期管理（创建、更新、退役）。
- **持久化**：agent 被周期性"nudge"把重要知识写入记忆——不是被动等用户保存。
- **回忆**：三层检索——SQLite + FTS5 全文本会话搜索（`hermes_state.py` SessionDB）、LLM 摘要辅助跨会话回忆、Honcho 方言式用户建模（dialectic user modeling，持续构建"你是谁"的画像）。
- **记忆后端可插拔**：`MemoryProvider` 抽象接口只有三个方法（`sync_turn` / `prefetch` / `shutdown`），官方插件已有 honcho / mem0 / supermemory / byterover / hindsight / holographic / openviking / retaindb 八个。

### 4.2 离线进化（GEPA，agent 下班之后）

运行时自学有两个已知致命伤，Nous 自己也不回避：**agent 倾向于对自己的表现过度乐观**（永远觉得自己干得不错），以及**自改进系统可能把你手写的 Skill 改得更差**。解法是把进化挪出运行时，做成独立管道（`NousResearch/hermes-agent-self-evolution`，ICLR 2026 Oral 的 GEPA 方法产品化）：

1. 读执行轨迹，从失败处出发（不看自我感觉）；
2. 生成评估数据集（Claude Opus 合成用例、真实会话历史、或人工黄金集）；
3. GEPA 优化器产出候选变体；
4. LLM 按评分标准打分（非 pass/fail 二元）；
5. 硬约束：测试套件 100% 通过、Skill ≤ 15KB、语义目的不漂移；
6. 最佳变体**作为 PR 提交，人类 review 后合并**——agent 没有直接 commit 权。

四个进化阶段分别针对 Skill 文本（`evolve_skill`）、工具描述（`evolve_tool_descriptions`）、系统提示词段（`evolve_prompt_section`）、工具代码（`evolve_tool_code`）。成本：纯 API 调用、无需 GPU，单次优化约 $2–10（xmsumi 拆解，2026-05-18）。

### 4.3 多 agent 与看板

- **角色系统**：`leaf`（默认，纯执行者，无 delegate_task / memory 等权限）与 `orchestrator`（可派生 workers）。硬上限：最大并发 3、嵌套深度 2、子 agent 超时 300 秒、默认不自动审批。
- **Kanban 看板（v0.13.0）**：持久化多工作流协作板，Board 是硬边界（worker 由环境变量钉死）、Tenant 是软命名空间；心跳保活、超时任务自动回收重分配、僵尸节点检测、重试预算、**幻觉门控**（worker 声明建卡需验证）、未完成即退出的 worker 自动阻塞。CLI 动词齐全：`hermes kanban init/create/assign/complete/block/heartbeat/stats/daemon`。
- **持久性纪律**：`delegate_task` 不跨轮次存活——长任务必须显式选 `cronjob` 或 `terminal(background=True, notify_on_complete=True)`。这是" ephemeral 协作 vs 持久任务"的正交设计，值得抄。

### 4.4 设计决策三问

1. **为什么进化走"运行时创建 + 离线精调"双轨，而不是全靠在线学习？** 在线快但无评估易飘；离线贵但有轨迹、有测试、有评分标准。快慢分层，是性能与可信的折中。
2. **为什么进化产出是 PR 而不是 commit？** 把"agent 自我修改"的最后一道闸门留给人类——这是信任问题不是工程问题。同类系统（含 OpenClaw 的 skill 市场）都应如此。
3. **为什么记忆后端抽象只有三个方法？** sync / prefetch / shutdown 覆盖了"写入、读出、清理"最小面，换来八个后端插件的接入自由。接口越小，生态越大。

## 五、组件模式拆解

**5.1 Skill 系统（程序性记忆）**
- 定位：经验固化成的 SOP，agentskills.io 开放格式。
- 何时用：重复工作流第三次出现时，它该已经在你的技能库里了。
- 实例：`/skills` 浏览、按名触发；内置 40+ 技能（MLOps、GitHub 工作流、科研）；Skills Hub 社区市场。

**5.2 记忆系统（陈述性记忆）**
- 定位：三层检索 + 可插拔后端。
- 何时用：跨周/跨月回忆"我们上次怎么处理的"。
- 实例：FTS5 秒回关键词；LLM 摘要处理模糊回忆；Honcho 建模用户偏好。八个第三方记忆插件任选。

**5.3 终端后端（执行环境）**
- 定位：agent 的手在哪——七种后端 local / Docker / SSH / Singularity / Modal / Daytona / Vercel Sandbox。
- 何时用：本地沙箱不够就上 Docker；要 serverless 零闲置成本用 Modal / Daytona（休眠唤醒）。
- 实例：$5 VPS 常驻 vs serverless 按需唤醒，同一套工具语义。

**5.4 消息网关（渠道接入）**
- 定位：单网关进程接 Telegram / Discord / Slack / WhatsApp / Signal / Email / CLI，语音备忘录转写、跨端对话连续。
- 何时用：通勤路上用 Telegram 指挥云端 agent 干活。
- 实例：`hermes gateway setup` 后全平台共享 slash 命令（/model /compress /skills /stop）。

**5.5 Nous Portal（一键全家桶）**
- 定位：一个订阅覆盖 300+ 模型 + 工具网关（web 搜索、生图、TTS、云浏览器）。
- 何时用：不想攒五家 API key 的人。
- 实例：`hermes setup --portal` OAuth 登录即全配好；工具按后端可逐项换成自带 key，非全有或全无。

**5.6 研究向工具链**
- 定位：批量轨迹生成（Tinker-Atropos）、轨迹压缩——为训练下一代 tool-calling 模型供数据。
- 何时用：研究团队拿它当数据工厂。
- 实例：这条暴露了其商业意图：agent 是入口，数据和模型迭代才是资产。

## 六、生态格局

- **本体**：NousResearch/hermes-agent（Python 3.11+，uv 包管理，SQLite/FTS5 会话库，Rich TUI）；核心 `AIAgent` 类约 1.2 万行（`run_agent.py`），`HermesCLI` 约 1.1 万行（`cli.py`）——单文件体量说明迭代速度优先于模块化（掘金架构解析，2026-05-14）。
- **衍生**：hermes-agent-self-evolution（GEPA 管道）、HermesClaw（社区微信桥，同一微信账号共跑 Hermes 与 OpenClaw）、computer-use-linux（Linux 桌面控制 MCP，AT-SPI 无障碍树 + Wayland/X11 输入）。
- **与 OpenClaw 的对位**（lynxflow 事实核对，2026-07-31）：Python vs Node.js；"自我进化单 agent" vs "多渠道网关"；"skills 从使用中自创建" vs "skills 社区市场"。渠道数 18 vs 24+，Hermes 国内渠道靠社区桥补。趋同信号明显：OpenClaw 在补 memory plugins，Hermes 在补渠道——一年后可能正面撞车。
- **商业面**：Nous Portal 订阅（模型 + 工具网关打包）是明牌变现；研究向轨迹工具是暗线资产。

## 七、对比评估

| 维度 | Hermes Agent | OpenClaw | 裸 coding agent |
|---|---|---|---|
| 核心叙事 | 越用越聪明（学习闭环） | 无处不在（渠道网关） | 单次任务 |
| 进化机制 | 运行时创建 + GEPA 离线精调 + PR 闸门 | 无内置进化，靠 skill 市场与人装 | 无 |
| 记忆 | 三层检索 + 8 可插拔后端 | Markdown 文件 + 语义搜索 | 视实现 |
| 多 agent | Kanban 协作板 + 角色 | 隔离路由 + delegate 治理 | 无 |
| 执行环境 | 7 终端后端含 serverless | 本机为主 + 沙箱策略 | 本机 |
| 渠道 | 约 18 适配器 | 24+ 含国内全家桶 | 无 |
| 迁移互操作 | `hermes claw migrate` 抢 OpenClaw 用户 | 2026.10.1-beta 支持导入 coding agent transcripts | — |

决策建议：单 agent 深度个性化场景（个人工作流沉淀）选 Hermes；多渠道接入与组织治理场景选 OpenClaw；两者都在快速互相抄作业，押注时把"哪边的架构演进你能影响"（社区贡献话语权）纳入考量。二手资料显示两者 MIT 互通、迁移成本低，先跑一个两个月再定是合理策略。

## 八、实践：最小路径

```bash
# Linux / macOS 一行装
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
source ~/.bashrc
hermes setup          # 向导：provider、工具、渠道一次配齐
hermes model          # 切模型（200+ 任意切换）
hermes                # 开聊
hermes gateway start  # 起渠道网关
```

验收标准：给它一件你重复干过三次的活，一周后 `/skills` 里应该出现它自己提炼的 Skill；故意纠正它一次，看 Skill 是否被修订。

使用纪律三条：定期 `git log` 你的 `~/.hermes/skills/`（审它写了什么）；对自动创建的 Skill 设预算（数量与体积）；GEPA 产出的 PR 必须跑过你自己的任务样本再合并。

## 九、失败模式与坑

1. **自进化失控型**：运行时自动创建 Skill 无人审，错误经验被固化为 SOP，越用越错且不自知。缓解：定期审计 skills 目录 + 开 GEPA 约束（测试 100%、≤15KB、语义不漂移）+ PR 闸门。（安全必答：agent 改写自身程序性知识，等同于可自我修改的代码——注入恶意指令的间接路径。）
2. **过度乐观型**：agent 自认为干得不错，实际轨迹一塌糊涂——运行时闭环的内生缺陷，Nous 用 GEPA"读轨迹不看自述"化解；不用离线管道的用户请降低对"自改进"的信任。
3. **覆盖人工技能型**：自动改进把你手调好的 Skill 改差。约束已内置（GEPA 语义漂移检查），但运行时轻量修订仍可能发生——手调 Skill 建议加只读保护。
4. **星数幻觉型**：各报道星数口径差异巨大（4.7 万至 16.7 万），二手数据以最新 GitHub 实数为准，别把中文二手报道的星数当决策依据。
5. **单文件巨型类型**：run_agent.py / cli.py 各逾万行，快速迭代的代价是内部耦合，二次开发门槛高于宣传，插件优先于改源码。
6. **Windows 误报型**：捆绑的 uv.exe 常被 Defender 误杀——官方给了哈希验证流程，按流程验明正身再加白名单（目录级，非文件级）。

## 十、结论与下一步

判断：**"越用越聪明"不是噱头，但也不是魔法——它是一套被工程约束住的进化机制。** 创建（运行时）与精调（离线 GEPA）分层、人类 PR 闸门、可插拔记忆后端，这三件设计让"自进化"从营销词变成了可审计的流水线。它是 2026 年"agent 个人化"方向最完整的开源实现。

下一步盯三个信号：GEPA 约束的社区实证（是否真守住语义不漂移）；与 OpenClaw 的功能趋同后谁的治理模型胜出；Nous Portal 订阅对第三方 provider 的中立性。

给不同读者的一句话：

- **开发者**：装一个，重复任务跑两周，然后打开 `~/.hermes/skills/` 看它给你写了什么——那是这个 agent 真正的产品。
- **架构师**：抄它的双轨进化（在线创建 + 离线精调）和 PR 闸门；这两件比任何单点功能都值得搬进你的系统。
- **产品经理**：注意 Nous 的打法——agent 免费、进化管道开源、模型与工具订阅收费，"能力沉淀在用户的 skills 里，收入沉淀在自己的 portal 里"，这是个人 agent 品类目前最清晰的商业模式。

## 附录：FAQ

1. **Hermes Agent 和 Hermes 模型什么关系？** 同公司（Nous Research）两个产品：模型是基座，agent 是框架。agent 可跑任意模型，模型可服务任意框架。
2. **"越用越聪明"有量化证据吗？** 官方叙事为主，独立基准尚缺；GEPA 有 ICLR 2026 Oral 论文背书其优化方法，但"长期个性化收益"仍待社区实证，本文建议以自用 skills 目录为真实评估口径。
3. **和 OpenClaw 能互迁吗？** 单向顺畅：Hermes 提供 `hermes claw migrate` 从 OpenClaw 导入记忆/技能/密钥/渠道；反向迁移工具未见，社区有 HermesClaw 桥让两者共享微信账号。
4. **安全吗？** 命令审批、DM pairing、容器隔离内置；最大风险面是自进化机制本身——Skill 审计与 PR 闸门不可省。
5. **国内渠道支持？** 飞书钉钉在适配列表里，微信靠社区桥（HermesClaw），生产使用前先验证桥接稳定性。

## 参考资料

### 一手（官方仓库 / 文档 / 进化管道）

1. [NousResearch/hermes-agent 主仓（README）](https://github.com/NousResearch/hermes-agent)
2. [NousResearch/hermes-agent-self-evolution（GEPA 管道与 PLAN.md）](https://github.com/NousResearch/hermes-agent-self-evolution)
3. [Hermes Agent 官方文档](https://hermes-agent.nousresearch.com/docs/)

### 二手（媒体 / 深度解析 / 对比）

4. [极客公园：两个月 4.7 万星，爆火的 Hermes Agent 是下一个龙虾（2026-04-11）](https://www.geekpark.net/news/362427)
5. [掘金：NousResearch/Hermes-Agent 深度技术解析（基于 v0.13.0，2026-05-14）](https://juejin.cn/post/7639583242592829490)
6. [xmsumi：三层记忆 + 自进化 Skill，GEPA 拆解（2026-05-18）](https://www.xmsumi.com/detail/3244)
7. [lynxflow：Hermes vs OpenClaw 事实核对（2026-07-31）](https://blog.lynxflow.co/posts/hermes-vs-openclaw/)
8. [阿里西西：$5 部署会自我进化的私人 Agent（2026-04-09）](https://www.alixixi.com/wz/382814.html)
9. [知乎专栏：Hermes Agent 自进化 AI 智能体（2026-02-25）](https://zhuanlan.zhihu.com/p/2042537202576512099)
10. [AgentGuide：Hermes Agent 自进化备战手册](https://github.com/adongwanai/AgentGuide/blob/main/docs/04-interview/18-agent-interview-playbooks/hermes-self-evolution-playbook.md)
11. [clauday：NousResearch 推出自我进化的开源 Agent 框架（2026-03-23）](https://clauday.com/zh/article/5f56e5c8-8cfd-437a-9c95-3d6b69dda486)
