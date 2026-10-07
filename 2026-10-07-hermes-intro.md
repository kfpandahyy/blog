# Hermes 是什么：一个开源大模型系列的简史

> 先说清楚：这篇文章讲的 Hermes，不是希腊神话里的信使神，也不是 React Native 里那个 JavaScript 引擎。它是 Nous Research 推出的一系列开源大语言模型。因为名字和奢侈品牌 Hermès 同词，中文社区索性叫它"爱马仕"——一个研究小组做的模型，被叫出了奢侈品的味道。

## 一、Hermes 的来历：一个研究集体的作品

Hermes 出自 **Nous Research**。这家机构 2022—2023 年间由一群开源社区出身的研究者组建，性质更接近研究集体而非传统公司。灵魂人物是 **Teknium**（Ryan），他在 LLaMA 发布之前就在开源社区发布微调模型，曾任职于 Stability AI，后来联合创立了 Nous，并一直亲自担任 Hermes 1 到 4 代的主要作者。

Nous 的自我定位是"开放模型、对齐用户"：不做封闭 API，模型权重全部开源，走"少过滤、高可控（minimally filtered, highly steerable）"的路线。2025 年，Nous Research 获得由 Paradigm 领投的 5000 万美元 A 轮融资，也做起了去中心化训练网络（Psyche）的探索。

## 二、发展脉络：四代模型，四个节点

**Hermes 1（2023）——起点。**
Teknium 基于 LLaMA 系列的早期微调作品。当时的社区还没有"开源 ChatGPT"的标杆，Hermes 是最早一批让人意识到"微调可以做出接近 GPT 的对话体验"的项目之一。

**Hermes 2 / OpenHermes-2.5（2023 年底）——成名。**
基于 Mistral 7B 微调，训练数据约一百万条，绝大部分由 GPT-4 生成。这是当年社区里最受欢迎的 7B 级对话模型之一，也奠定了 Hermes "数据合成 + 后训练"的方法论。

**Hermes 2 Pro（2024 年 3—5 月）——Agent 化的关键一步。**
这一代最重要的贡献不是模型本身，而是 **Hermes Function-Calling 标准**：工具定义放在 `<tools>` 标签里，调用写在 `<tool_call>` 里，执行结果回填 `<tool_response>`——这些特殊标签被设计为独立 token，让流式输出中的工具调用可以被可靠解析。在闭源 API 之外，这是开源世界第一次有模有样地复制出函数调用能力。配套开源的 `hermes-function-calling-v1` 数据集至今仍是训练工具调用能力的常用素材（下载量超 26 万）。官方数据：函数调用评测 90 分，结构化 JSON 输出 84 分（与 Fireworks.AI 合作评测）。

**Hermes 3（2024 年 8 月）——触及开源前沿。**
基于 Llama 3.1 的 8B、70B、**405B** 三档微调，是 Llama 3.1 405B 发布后的第一个微调版本。技术报告（arXiv:2408.11857）里的关键词是"**高度可控（highly steerable）**"：模型对系统提示极其敏感，愿意按照 system prompt 设定的人设和世界观回应，而不是默认摆出"乐于助人助手"的姿态。405B 版本尤为明显——空系统提示下它不一定会进入助手人格。

这一代还引入了面向智能体的改进：在直接输出 `<tool_call>` 之前，先在一个推理块里显式复述目标、规划动作、反思结果（社区常称 GOAP 式推理），显著减少了无效工具调用。训练工程上也有看点：405B 用 128 张 H100，配合 Axolotl 框架和 FSDP，最终用 FP8 量化把显存和磁盘需求压低约一半，让 405B 能在单节点跑起来。

**Hermes 4（2025 年 8 月）——混合推理。**
延续 405B/70B（Llama 3.1 底座），14B 小模型改用 Qwen3 底座。最大变化是加入**混合推理模式**：模型自主决定是否在 `<think>` 段里先做显式推理，再给出回答。在对"拒答率"的评测（RefusalBench）上，Hermes 4 的拒答率为同规模主流模型中最低——比 GPT-4o、Grok 4、DeepSeek V3 都更愿意回应，只在少数高危类别（未成年人伤害、人口贩卖、自伤）上保留了拒绝逻辑。这很符合 Nous 一贯的立场：对齐用户，而不是对齐审查。

## 三、Hermes 的技术标签

把四代的共性抽出来，Hermes 系列的辨识度在三点：

1. **高可控性**。系统提示即人设，模型忠实地"进入角色"，角色扮演、创意写作是长项；代价是对 prompt 工程更敏感，空 prompt 行为不可预测。
2. **工具调用标准输出**。`<tool_call>`/`<tool_response>` 的 XML 风格协议先于业界 OpenAI function calling 格式在开源圈流行，大量开源 Agent 框架的早期实现都参考了它。
3. **数据合成驱动的后训练**。从 OpenHermes 到 Hermes 3/4，训练数据几乎完全由合成数据流水线生成，这是 Nous 方法论的核心，也影响了后来一批开源微调项目。

## 四、业界位置与争议

客观说：Hermes 不是每个时代的榜首。Hermes 3 405B 发布时是开放权重的第一梯队，但很快被 Tülu 3、DeepSeek 系列等超越；它的单项 benchmark 很少霸榜。它的历史地位更多体现在**范式贡献**上：

- 证明了"微调 + 合成数据"能让开源模型逼近闭源对话体验（Hermes 2 时代）；
- 给开源 Agent 生态提供了第一套可复用的函数调用协议（Hermes 2 Pro）；
- 把"模型该对齐谁"的问题摆上台面——Hermes 的低拒答路线在社区拥趸众多，也被认为需要使用者自行承担更多安全责任。

中文社区给它起的"爱马仕"外号半是玩笑半是敬意：一个十来人的小团队，连续四年站在开源微调的第一线，这本身就是稀缺品。

## 参考资料

1. Hermes 3 Technical Report (Teknium, Quesnelle, Guang, 2024) — arXiv:2408.11857，https://arxiv.org/abs/2408.11857
2. Hermes 4 Technical Report (Teknium et al., 2025) — arXiv:2508.18255，https://arxiv.org/abs/2508.18255
3. NousResearch/Hermes-2-Pro-Mistral-7B 模型卡 — https://huggingface.co/NousResearch/Hermes-2-Pro-Mistral-7B
4. Hermes Function-Calling 代码仓库 — https://github.com/NousResearch/Hermes-Function-Calling
5. hermes-function-calling-v1 数据集 — https://huggingface.co/datasets/NousResearch/hermes-function-calling-v1
6. A Brief History of Agents（Hermes 系列在 Agent 史上的定位评述）— https://vectorculture.substack.com/p/a-brief-history-of-agents
7. 新智元：10 人明星团队炼出首个微调 Llama 3.1 405B — https://m.thepaper.cn/newsDetail_forward_28422428
8. AI2 Tülu 3 后训练论文（含与 Hermes 3 405B 的对比）— arXiv:2411.15124
9. SiliconANGLE：Nous Research 获 Paradigm 领投 5000 万美元 — https://www.siliconangle.com/2025/02/05/paradigm-leads-50m-round-decentralized-ai-project-nous-research/
10. Hermes Agent 深度研究报告（中文）— https://www.zhangfeibiao.com/archives/Hermes-Agent

---

*写于 2026-10-07。模型迭代很快，发布半年内的信息以官方渠道为准。*
