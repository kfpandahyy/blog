# Hermes 是什么：一个开源大模型系列的简史

> **TL;DR**：Hermes 是 Nous Research 基于 LLaMA/Mistral/Qwen 等开源底座微调的一系列对话模型，卖点是高可控性、工具调用协议和合成数据后训练。它很少霸榜，但三次推动开源社区范式：证明"微调+合成数据"能逼近闭源体验、给开源 Agent 生态第一套函数调用标准、把"模型对齐谁"的问题摆上台面。行动建议：学 Agent 工程的人该读它的 `<tool_call>` 协议血统；选型的人注意它"低拒答、高可控"是能力也是责任转移。

---

中文社区管 Hermes 叫"爱马仕"——一个十来人的研究小组做的开源模型，被叫出了奢侈品牌的味道。这个外号半是玩笑半是敬意：从 2023 到 2025，这个小团队连续四代站在开源微调的第一线。

本文的核心判断：**Hermes 的历史地位不在 benchmark 榜首，在范式贡献**。它单项测评很少第一，却三次改变了开源社区的做法。本文先讲清它是什么，再按时间线走四代，然后解剖它最重要的技术遗产——函数调用协议，最后给选型判断和已知的坑。

理解 Hermes 最好的类比是**改装车厂**：Nous 不造发动机（基座模型来自 Meta/Mistral/阿里），它买的是大厂底盘，调校出自己的性格——更听话、更会用车载工具、更少拒绝车主。

---

## 一、定义与边界

**一句话定义**：Hermes 是 Nous Research 基于开源基座模型微调的一系列对话大模型，以高可控性（system prompt 即人设）、XML 风格的工具调用协议、合成数据驱动的后训练为三大技术标识。

最小实例——它最著名的技术遗产，函数调用的真实输出格式（Hermes 2 Pro 起）：

```xml
<tools>
{"type": "function", "function": {"name": "get_weather", ...}}
</tools>

<tool_call>
{"name": "get_weather", "arguments": {"city": "北京"}}
</tool_call>

<tool_response>
{"temperature": "18C", "condition": "晴"}
</tool_response>
```

这些特殊标签被设计为独立 token，让流式输出中的工具调用可以被可靠解析——在闭源 API 之外，这是开源世界第一次有模有样地复制出函数调用能力。

**is / is-not**：

- **不是基座模型**：底座始终是别人的（LLaMA → Mistral → Llama 3.1 → Qwen3），Hermes 的价值在微调与后训练。
- **不是 Hermès**：和奢侈品牌同词，纯属巧合。
- **不是 Hermes Agent**：那是 Nous 后来做的 Agent 框架（属于 harness，见本博客《Agent Harness 是什么》），模型系列是另一回事。

**与相邻路线辨析**：

| 路线 | 代表 | 与 Hermes 的区别 |
|---|---|---|
| 基座模型 | LLaMA、Qwen、Mistral 原版 | Hermes 是其二次微调，主打对话与可控性 |
| 学术微调标杆 | AI2 Tülu 3 | 方法论相近（合成数据后训练），Tülu 更偏评测驱动，Hermes 更偏对话体验与 Agent 能力 |
| 自研路线 | DeepSeek 系列 | 从头训练 vs 微调，成本量级不同，Hermes 是后者的代表 |

---

## 二、何时用，何时不用

**适用**：

1. 研究工具调用协议的设计——它是开源函数调用格式的鼻祖，读它的协议演进胜过读十篇综述。
2. 需要本地可跑、行为可控的对话模型（角色扮演、创意写作、风格敏感的客服类场景）。
3. 研究后训练方法论——合成数据流水线是它的核心手艺。

**不适用**：

1. 追求 benchmark 榜首——Hermes 从来不是这条路线的玩家，发布半年内被超越是常态。
2. 需要官方强安全护栏的对外产品——它的低拒答立场意味着安全责任转移给使用者，合规审查会难过。
3. 需要长上下文、多模态等基座能力上限的场景——微调不改架构，底座有什么它才有什么。

**Trade-off 说透**：高可控性是双刃剑。系统提示即人设 = 对 prompt 工程极敏感，空 system prompt 下行为不可预测（Hermes 3 405B 甚至不默认进入助手人格）。换来的是同规模主流模型中几乎最低的拒答率。

---

## 三、演进脉络

```
2022-2023  Nous Research 组建：开源社区研究者联盟，灵魂人物 Teknium
2023       Hermes 1：基于 LLaMA 早期微调，"开源 ChatGPT" 概念萌芽
2023 年底  Hermes 2 / OpenHermes-2.5：基于 Mistral 7B，约 100 万条 GPT-4
           合成数据训练，社区最受欢迎的 7B 对话模型之一
2024-03/05 Hermes 2 Pro：发布 <tool_call> 函数调用标准 + 开源数据集
           （hermes-function-calling-v1，下载量超 26 万）
2024-08    Hermes 3：Llama 3.1 8B/70B/405B 三档，405B 发布后的首个微调版；
           引入 GOAP 式推理块；128 张 H100 + FSDP + FP8 训练工程
2025-02    Nous 获 Paradigm 领投 5000 万美元 A 轮
2025-08    Hermes 4：混合推理（自主决定是否 <think>）；RefusalBench 上
           拒答率为同规模主流模型最低；14B 改用 Qwen3 底座
```

路径选择：同期开源社区有两条路——自研基座（DeepSeek 路线，烧钱换上限）或深度微调（Nous 路线，轻资产换迭代速度）。Nous 用四代产品证明后者在"快速跟进 + 范式探索"上有效率优势，代价是永远跟在基座发布之后。

---

## 四、解剖：技术遗产的三个剖面

以 Hermes 2 Pro 的函数调用为实例，从协议设计讲到训练工程。

### 4.1 函数调用协议：为什么这样设计

协议三要素：`</tools>` 定义可用工具，`<tool_call>` 发起调用，`<tool_response>` 回填结果。关键设计决策：

1. **为什么用 XML 标签而不是 JSON 裸串**：JSON 在流式生成中无法可靠判断"这一段是正文还是调用"——生成到一半无法解析。独立 token 的闭合标签让解析器可以边生成边判断，流式场景下零歧义。
2. **为什么标签设计为独立 token**：进入词表成为原子单位，模型不会在标签内部"自由发挥"出畸形的半个标签。
3. **为什么在开源世界先于 OpenAI 格式流行**：OpenAI function calling 是闭源 API 能力，开源社区需要可自训、可魔改的开放协议。大量开源 Agent 框架的早期实现参考了它——这是 Hermes 影响最深远的遗产。

官方评测（与 Fireworks.AI 合作）：函数调用 90 分，结构化 JSON 输出 84 分（2024）。

### 4.2 Hermes 3 的 Agent 化改进

在直接输出 `<tool_call>` 之前，先在一个推理块里显式复述目标、规划动作、反思结果（社区称 GOAP 式推理）。效果：显著减少无效工具调用。这是"让模型先想再做"在开源世界的早期系统化实践，比 Reasoning 模型风潮早几个月。

训练工程的看点：405B 用 128 张 H100，Axolotl 框架 + FSDP，FP8 量化把显存和磁盘需求压低约一半，让 405B 能在单节点跑起来——这对"超大模型微调成本"是个可参考的工程样本。

### 4.3 合成数据流水线：方法论核心

从 OpenHermes 到 Hermes 3/4，训练数据几乎完全由合成数据流水线生成（GPT-4 生成 → 过滤 → 配比）。三个决策：

1. **为什么敢用模型生成的数据训模型**：2023 年"模型自我污染"的质疑很盛，Nous 用两代产品的质量证明高质量合成数据 + 严格过滤可逼近蒸馏效果。
2. **为什么是方法论而非秘方**：数据集全部公开（hermes-function-calling-v1 下载量 26 万+），配方在模型卡里——开放本身就是影响力策略。
3. **代价**：合成数据的分布偏置（过度礼貌、风格趋同）会遗传给模型，高拒答模型的输出痕迹也需要清洗。这后来成为所有合成数据路线的共性问题。

---

## 五、四代模型拆解

统一微结构看四代（是什么 / 历史角色 / 关键技术 / 注意）：

### 5.1 Hermes 1（2023）

- **是什么**：基于 LLaMA 的早期微调对话模型。
- **历史角色**：起点，"微调可以做出接近 GPT 的对话体验"的早期证明。
- **关键技术**：常规指令微调，谈不上方法论。
- **注意**：纯历史价值，今天不要碰。

### 5.2 Hermes 2 / OpenHermes-2.5（2023 年底）

- **是什么**：基于 Mistral 7B，约 100 万条 GPT-4 合成数据。
- **历史角色**：社区最受欢迎的 7B 对话模型之一，奠定"数据合成 + 后训练"方法论。
- **关键技术**：大规模合成数据配比。
- **注意**：底座老旧，仅具考古价值。

### 5.3 Hermes 2 Pro（2024-03/05）

- **是什么**：7B 模型 + 函数调用能力的打包发布。
- **历史角色**：开源函数调用标准的源头（本文第四章的主角）。
- **关键技术**：`<tool_call>` 协议 + 独立 token + 配套开源数据集。
- **注意**：协议比模型长寿——今天学协议价值大于用模型价值。

### 5.4 Hermes 3（2024-08）

- **是什么**：Llama 3.1 8B/70B/405B 三档微调。
- **历史角色**：405B 发布后的首个微调版，触及开源前沿；"高度可控"标签的来源。
- **关键技术**：GOAP 推理块；128×H100 + FSDP + FP8 工程。
- **注意**：405B 空系统提示不默认进入助手人格——可控性是把双刃剑。

### 5.5 Hermes 4（2025-08）

- **是什么**：混合推理模型，自主决定是否在 `<think>` 段先推理再回答。
- **历史角色**：拒答率评测（RefusalBench）上为同规模主流模型最低，把"对齐用户"路线推到极致。
- **关键技术**：混合推理开关；14B 档换用 Qwen3 底座（务实路线的信号——谁底座好用谁）。
- **注意**：低拒答 = 安全责任在使用者侧，见下章。

---

## 六、生态格局

Nous Research 全景（截至 2026 年中）：

| 资产 | 一句话定位 | 状态 |
|---|---|---|
| Hermes 模型系列 | 高可控开源微调模型 | 活跃，四代 |
| Hermes Function-Calling 标准 + 数据集 | 开源函数调用格式鼻祖 | 事实标准遗产，下载 26 万+ |
| Hermes Agent | Agent 框架（harness 层） | 活跃，与模型系列独立 |
| Psyche | 去中心化训练网络探索 | 融资后新项目 |
| 公司 | Paradigm 领投 5000 万美元 A 轮（2025-02） | 约十来人团队 |

---

## 七、对比评估

### Hermes 路线 vs 自研路线

| 维度 | Nous（深度微调） | DeepSeek（自研基座） |
|---|---|---|
| 成本结构 | 轻资产，跟基座迭代 | 重资产，烧钱换上限 |
| 能力上限 | 受底座约束 | 自主定义 |
| 迭代速度 | 基座发布后数周内跟进 | 自主节奏 |
| 范式贡献方式 | 协议、方法论、数据集 | 架构、Scaling 经验 |

**决策建议**：研究 Agent 协议与后训练方法论 → 深读 Hermes；选型基座 → 直接看最新一代开源基座（Qwen/DeepSeek/Llama），Hermes 是观察"微调能加多少分"的样本，不是默认选项。

---

## 八、实践：上手与最佳实践

**最小路径**（研究协议，不需要训练）：

```bash
# 跑一个 Hermes 2 Pro 体验原始函数调用格式
ollama run nous-hermes2-pro  # 或从 HuggingFace 下载 GGUF
```

**最佳实践**：

1. 学协议：读 `NousResearch/Hermes-Function-Calling` 仓库的 README 与示例，比读二手综述快。
2. 用数据集：`hermes-function-calling-v1` 至今是训练工具调用能力的常用素材，做自己的微调任务可以直接借用其格式。
3. 评估可控性：测 Hermes 类模型时，把 system prompt 敏感度当一等指标——空 prompt、模糊 prompt、对抗 prompt 各跑一组。
4. 追新不追旧：要用就用 Hermes 4，老代仅作历史对照。

---

## 九、失败模式与坑

1. **高可控 = 高 prompt 敏感**。怎么发现：同一任务换 system prompt 措辞，输出风格剧变甚至人格漂移。怎么绕：生产环境锁死 prompt 模板；把空 prompt 行为纳入测试集。
2. **低拒答的安全责任转移**。Hermes 只保留极少数高危类别（未成年人伤害、人口贩卖、自伤）的拒绝逻辑。怎么发现：合规审查。怎么绕：对外产品必须自行加护栏层——harness 的 guardrails（见本博客《Agent Harness 是什么》），不能指望模型自觉。
3. **被超越快**。怎么发现：发布后数月内被 Tülu 3、DeepSeek 等在单项超越。怎么绕：把 Hermes 当方法论来源，不当长期选型。
4. **合成数据偏置遗传**。输出过度礼貌、风格趋同。怎么绕：做后处理与风格重校准；混合真实人类数据。

---

## 十、结论与下一步

三行收束：Hermes 是改装车厂式的开源微调路线代表；函数调用协议与合成数据方法论是它留给行业的两份长期资产；"高可控、低拒答"是设计立场，使用即接受责任转移。

**给不同读者的下一步**：

- **开发者**：读 Hermes-Function-Calling 仓库两小时，理解 `<tool_call>` 设计取舍——你在任何框架里写工具调用都在继承这份遗产。
- **团队负责人**：评估开源对话模型时，把"prompt 敏感度"和"拒答策略"列为选型指标，不只看 benchmark。
- **产品经理**：Nous 是"小团队靠方法论与开放策略获得行业影响力"的样本——数据集与协议全开放反而成了它的护城河，这个逻辑值得抄。

---

## 附录：FAQ

**Q1：Hermes 和爱马仕有关系吗？**
没有。名字撞车 Hermès，中文社区的"爱马仕"外号纯属玩笑，注意拼写也不同（Hermes vs Hermès）。

**Q2：Hermes Agent 和 Hermes 模型是什么关系？**
同一个团队的两条产品线：模型是大脑，Hermes Agent 是套在模型外面的 harness（脚手架）。类比本博客《Agent Harness 是什么》里的分层即可。

**Q3：现在做 Agent 还用 Hermes 2 Pro 吗？**
模型本身老了，但它的协议设计和数据集仍在被继承。学协议、借数据集，不用模型。

**Q4：Hermes 3 405B 个人能跑吗？**
不能。405B 需要数据中心级显存。个人体验选 7B/8B/14B 档，或看 GGUF 量化版在本地推理框架上的表现。

---

## 参考资料

### 一手（技术报告 / 模型卡 / 仓库）

1. Hermes 3 Technical Report — arXiv:2408.11857，https://arxiv.org/abs/2408.11857
2. Hermes 4 Technical Report — arXiv:2508.18255，https://arxiv.org/abs/2508.18255
3. NousResearch/Hermes-2-Pro-Mistral-7B 模型卡 — https://huggingface.co/NousResearch/Hermes-2-Pro-Mistral-7B
4. Hermes Function-Calling 仓库 — https://github.com/NousResearch/Hermes-Function-Calling
5. hermes-function-calling-v1 数据集 — https://huggingface.co/datasets/NousResearch/hermes-function-calling-v1

### 二手（解读 / 媒体 / 综述）

6. Vector Culture: A Brief History of Agents — https://vectorculture.substack.com/p/a-brief-history-of-agents
7. 新智元：10 人明星团队炼出首个微调 Llama 3.1 405B — https://m.thepaper.cn/newsDetail_forward_28422428
8. AI2 Tülu 3 后训练论文（含与 Hermes 3 405B 对比）— arXiv:2411.15124
9. SiliconANGLE：Paradigm 领投 Nous Research 5000 万美元 — https://www.siliconangle.com/2025/02/05/paradigm-leads-50m-round-decentralized-ai-project-nous-research/
10. Hermes Agent 深度研究报告（中文）— https://www.zhangfeibiao.com/archives/Hermes-Agent
