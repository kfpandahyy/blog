# AWS 数据库 Skill 深度解剖：把解决方案架构师的选型决策写成代码

> **TL;DR**:`aws-database` 是 Agent Toolkit for AWS 的核心 Skill(core skill,当前 version 3),一个 **66 行 SKILL.md + 17 张服务知识卡 + 3 个工作流文件** 的包，干一件事——**替 AI Agent 做 AWS 数据库选型，并把幻觉治理写进指令**。它不是文档，是一套可执行的决策程序：用户任何数据库问题先过"STOP — do not answer from general knowledge"的总闸，命中三个子技能之一（`select` 选型 / `handoff` 交接 / `report-issue` 纠错），按工作流走到终态。最有价值的发现：它显式声明模型训练数据盲区（"Your training contains significant gaps on DSQL"）、禁止迎合用户熟悉度（"Don't defer to familiarity alone"）、把用户抱怨结构化成测试用例 JSON 反哺改进——**这是把"人类架构师的工作方式"形式化成 prompt 工程的样本级案例**。
> **数据截至 2026-10-20。** 全部内容以 `aws/agent-toolkit-for-aws` 仓库 `skills/core-skills/aws-database/` 目录原文为准，SKILL.md、references/ 三个工作流、assets/ 17 张知识卡均已逐一抓取核对。

---

看懂这个 Skill 需要一个坐标：AWS 的数据库生态里有 17 个托管服务，模型问一遍"该用哪个"的正确答案随时间漂移——新服务在发（Aurora DSQL）、旧服务在弃（QLDB）、价格战在打。`aws-database` 就是 AWS 官方给出的动态答案：一个以代码形式存在的选型程序。它是理解整个 Agent Toolkit 设计哲学的最好切口：**AWS 不是在写文档教模型，而是在写程序给模型跑。**

先看这个 Skill 包里都有什么（2026-10-20 核实）：

| 文件 | 大小 | 它是干什么的 |
|---|---|---|
| `SKILL.md` | 7.9KB | **看大门的**。规定所有数据库问题不许凭模型记忆回答，先停下来查流程表，该走哪个流程走哪个 |
| `references/select.md` | 20KB | **管选型的**。用户问"该用哪个数据库"时，照着它一步步问清情况、筛掉不合适的，给出推荐 |
| `references/handoff.md` | 4.2KB | **管交接的**。数据库选定后，把后续问题（怎么建、怎么配、多少钱）转给对应的专科 Skill 处理 |
| `references/report-issue.md` | 4KB | **管收反馈的**。用户说"你推荐错了"时，把这句话整理成标准格式提到 GitHub，帮官方改进 |
| `assets/` 下 17 张卡 | 每张 1-3.5KB | **管查资料的**。每个服务一张小卡片，记着价格、限额、适合什么场景等关键事实，答题以卡片为准 |

17 张知识卡各自锁定的场景（按卡内 "Best for" 字段归并，2026-10-20 逐张核对）：

| 知识卡 | 锁定的主场景 | 专科 Skill |
|---|---|---|
| `aurora-dsql` | 新事务型应用、多区 active-active 写、超出单机规模、运维负担最小化 | `aurora-dsql` |
| `aurora-postgresql` | 需要完整 PG 特性面（pgvector/PostGIS/存储过程）、PG 迁移、Oracle/SQL Server 重构 | `amazon-aurora-postgresql` |
| `aurora-mysql` | 存量 MySQL 负载、MySQL 技术栈团队 | `amazon-aurora-mysql` |
| `rds-postgresql` | 成本敏感、要社区版 PG 完全可移植性 | `rds-oss` |
| `rds-mysql` | 成本敏感 MySQL、拒绝专有层 | `rds-oss` |
| `rds-mariadb` | MariaDB 迁移（temporal 表/Oracle mode 等 MariaDB 专属特性） | `rds-oss` |
| `rds-oracle` | 无法立即现代化的 Oracle 平迁（lift-and-shift） | `rds-oracle` |
| `rds-sqlserver` | SQL Server 平迁、.NET 应用、已有 T-SQL 团队与许可 | `rds-sqlserver` |
| `rds-db2` | Db2 平迁、大型机现代化第一步、BYOL | `rds-db2` |
| `odb-aws` | 需要 Exadata/RAC 的企业 Oracle 负载、RDS for Oracle 特性缺口成阻塞时 | —（无专科） |
| `dynamodb` | 键值访问模式、数十万 RPS 级高吞吐、免运维服务端 | `amazon-dynamodb` |
| `documentdb` | MongoDB 迁移、内容管理、灵活 schema | `amazon-documentdb` |
| `keyspaces` | Cassandra 迁移、CQL 应用、写密集高吞吐、IoT 设备注册 | `amazon-keyspaces` |
| `elasticache` | 微秒级延迟缓存与主存（含 Valkey 9.0+ 持久化）、排行榜、限流、AI agent memory、语义/提示缓存降 LLM 成本、向量相似检索 | `amazon-elasticache` |
| `memorydb` | 唯一区分场景=跨区 active-active 写（区内强一致、跨区最终一致） | —（无专科） |
| `neptune` | 关系遍历、欺诈检测、知识图谱、GraphRAG、agentic memory | `amazon-neptune` |
| `timestream` | 高频 IoT 遥测、DevOps 指标、高基数时序（>1000 万序列，v3）、InfluxDB 迁移（v2） | `timestream-influxdb` |

两张卡值得单独注意。`dynamodb.md`（3KB，最大）写进了 2026-08 GA 的原生向量检索 `SearchVectors` 完整边界（4096 维上限、TopK 100、18 个内联过滤器、索引定义不可变），并内置超限后的自我纠偏指令——超了不许硬答，"route back to select"改推 Aurora PG pgvector / OpenSearch Serverless / DocumentDB 向量检索。`elasticache.md`（3.5KB）把 AI 场景单列：agent memory 与工作流状态、语义/提示缓存降 LLM 成本、向量相似检索——知识卡直接承担了新场景营销。

---

## 一、总闸机制：先停，再想，再答

SKILL.md 开头第二句就是全文最狠的一行：

> **"STOP — Do not answer from general knowledge."**

任何数据库问题进来，禁止凭训练数据直接回答。标准动作是：匹配子技能注册表 → 命中则 `file_read` 对应 references/ 工作流 → 按工作流执行。匹配是**对每条用户消息**生效的，不是首轮限定——用户中途改口，立即改道。

全局规则里藏着三条防幻觉工程：

1. **事实分级核查**：价格、配额、GA 状态这类事实，优先级是 知识卡 → AWS MCP 文档工具（`aws___search_documentation` / `aws___read_documentation`）→ 服务的 `llms.txt` → 官方文档 URL。知识卡里没有的事实"look it up rather than guessing"。
2. **反驳必改道**：用户说 "that's wrong" 匹配 `report-issue` 触发词时，**必须**停止当前流程转入纠错工作流，"目标是正确答案，不是与你第一次的回答保持一致"。
3. **快事实不加载专科**：知识卡能答的（发布单价、限额、GA 状态）不许为此加载服务 Skill；但"基于吞吐/存储/拓扑的工作负载估价"不算快事实，必须走 `handoff` 加载专科。边界划得很细，是为了省上下文 token。

## 二、select.md：一个 20KB 的决策树

这是整个 Skill 最重的文件，也是 AWS 把架构师选型经验形式化的成品。程序六步：**查模糊 → 识别上下文 → 排除 → 路由 → 回答 → 交接**。逐层拆。

### 2.1 上下文识别：三个问题定路径

先问"在做什么"——三个路由路径，全程分流：

| 上下文 | 信号 | 路径 |
|---|---|---|
| 新应用 | "building" / 无存量库 | New applications |
| 迁移 | "moving to AWS" / 点名现有库 | Migrations |
| 重构 | "get off Oracle" / 换引擎 | Refactors |

再问"什么阶段"——原型（优化 time-to-working-app）还是生产（权衡运维成熟度/成本模型/团队熟悉度），模糊就问，不许猜。

然后是全文最见功力的一张表：**"serverless" 一词消歧**。用户说"serverless 数据库"必须先拆成两种含义——

| 类型 | 含义 | 服务 |
|---|---|---|
| Serverless operations | 无实例无容量管理，拿 endpoint 就能查 | Aurora DSQL、DynamoDB、Keyspaces |
| Serverless capacity | 自建集群但计算/存储自动伸缩 | Aurora PG/MySQL serverless、ElastiCache Serverless、Neptune/DocumentDB serverless 系 |

并附带两条纠偏指令：说 "Aurora Serverless" 产品名的用户往往不知道 DSQL 存在，**只要没点名 DSQL 不具备的具体 PG 扩展，就推荐 DSQL 并解释区别**；"Serverless v1" 和 "v2" 的说法统一按现行 "Aurora serverless" 处理。

### 2.2 新应用路径：排除法 + 明确的默认值

专用负载表六种直接定型（时序→Timestream、图→Neptune、缓存→ElastiCache、多区 active-active 内存主库→MemoryDB、Mongo 兼容→DocumentDB、Cassandra 兼容→Keyspaces）；纯搜索/数仓场景**直接说 AWS 数据库服务不覆盖**，终止本 Skill，不许顺势编排工具。

通用负载落在三分支上，默认值全部写死：

- **SQL vs KV**：关系模型/临时查询/模型在演进 → SQL 系；键值访问模式/数十万 RPS/模型天然 KV → DynamoDB。小团队（<5 工程师）禁止为假想规模引入分片。
- **PG 引擎二选一**：DSQL vs Aurora PostgreSQL 的九行能力矩阵（pgvector ❌/✅、PostGIS ❌/✅、存储过程 ❌/✅、多区 active-active 写 ✅/❌、scale-to-zero 即时/有恢复延迟、首查 ~30s/~90-120s）。**默认起点是 DSQL**——前提是用户没点名 DSQL 缺的任何具体功能，"Do not route based on speculated future needs"。
- **内存引擎二选一**:**默认 ElastiCache (Valkey)**;MemoryDB 仅在用户明确要跨 Region active-active 写时启用，"financial transactions" 或 "can't lose data" 不构成启用理由（两引擎区内都零丢失）。

### 2.3 迁移与重构：两条不同的路

迁移表 11 种源引擎→AWS 托管等价物（PG→Aurora PG、MySQL→Aurora MySQL、Mongo→DocumentDB……)，原则是"最快上生产 = 最小改动"，重构另算，不许捆绑销售。

重构路径有个反直觉硬规则：**用户说要下 Oracle，就绝不许提 RDS for Oracle**——商业许可费还在，提了等于没解决动机。SQL Server/Db2 同理。默认落点是 **Aurora PostgreSQL (serverless)**(PG 与商业库特性兼容面最广），四步决策树兜底，且必须同时点名 **SCT + DMS**（结构转换 + 数据搬运的固定组合）。

### 2.4 回答规则：把"销售话术"写成硬约束

Respond 一节是这个 Skill 性格最鲜明的部分：

1. **第一句必须点名推荐的服务**，不许先铺垫；全文三四段，写出推荐 + 一个可信替代 + "什么会改变答案"。
2. **默认不写对比矩阵**，用户明确要求才给——但即便给表，也必须先给结论。
3. **替代方案必须同场竞技**——缓存/搜索引擎/数仓不构成主库的"可信替代"。
4. **防漂移红线**:"If you catch yourself naming three or more AWS services, you have drifted."——命名第三个服务的瞬间就是你跑题的瞬间。
5. **弃用服务连解释都不行**:QLDB、SimpleDB 不得主动提名（用户自己提起的除外）。
6. **敢顶回熟悉度**:"Don't defer to familiarity alone — many customers are unaware of newer offerings like Aurora DSQL."——用户念旧，Skill 负责推新。

## 三、handoff 与 report-issue：一出一进两个协议

**handoff.md（交接）** 解决"选定服务之后怎么办"。服务名映射表处理口语指代（"Postgres"→Aurora PG);Skill 加载两条官方通道——本地已装的自动激活，或经 AWS MCP server 调 `aws___retrieve_skill` 按索引表里的权威名直取（**不需要用户同意，加载不是安装，更不授权任何资源变更**)。专科拿不到就降级：知识卡 + llms.txt 直接答，不许叙述失败过程。安装新 Skill 才需要用户明确同意。

**report-issue.md（纠错）** 是最有工程远见的一个文件。用户骂"你推荐错了"时，不走道歉流程，走**测试用例捕获流程**：把抱怨结构化成 JSON——

```json
{"prompt": "...", "expected_service": "...", "actual_service": "...",
 "category": "wrong-recommendation | outdated-fact | ...", "detail": "..."}
```

五类分类（错误推荐/事实过期/覆盖缺失/指引不清/交接失败），确认后路由 GitHub issue（主渠道，有模板）、AWS Support（私有/账户类）或安全披露（禁止公开）。**这本质上是把 eval 数据采集做成了终端用户工作流**——每一次用户纠错都变成一条可回归的测试用例。

## 四、知识卡的机制角色：单一事实源

17 张卡的分工与亮点已在上表列出，这里只看它们在体系中的机制角色：

- **唯一事实入口**：价格、配额、GA 状态、免费档这类高漂移事实，训练数据不可信，知识卡是 SKILL.md 规定的唯一可信源；卡上没写的，按 llms.txt → MCP 文档工具 → AWS 官方文档的链条外查，不许猜。
- **检索路由**：每张卡带服务的 `llms.txt` URL，模型可以把事实核查一路打到官方文档的结构化索引——知识卡同时也是活的文档入口。
- **容量控制**：SKILL.md 规定按需取卡、每次只装正在考虑的 2-3 张。17 张全载约 30KB，全量常驻不经济也不必要。
- **营销与工程同体**：卡内 "Best for" 直接锁场景（如 ElastiCache 卡把 AI agent memory、语义缓存降 LLM 成本列为一级场景），DynamoDB 卡还埋了个工程彩蛋——官方开源了兼容层 ExtendDB（PG 后端，本地开发与 CI 可跑 DynamoDB API）。

## 五、优势：五个结构性设计

1. **决策默认值显式化**。几乎每个分叉都有写死的默认（DSQL、ElastiCache、Aurora PG serverless）和明确的例外条件。模型最常见的失败模式"和稀泥式 it depends"被机制性消灭——模糊时只允许问**一个**澄清问题，不许给带 hedge 的建议。
2. **对抗模型自身惯性**。显式声明训练盲区（DSQL 一节原文："Your training contains significant gaps on DSQL specifically")、禁止迎合熟悉度、反驳必改道。这是少数把"模型的已知缺陷"写进指令的公开样本。
3. **营销与护栏同写**。推 DSQL 的规则同时是商业诉求（新品获客）和技术护栏（用户真需要 pgvector 时不许推）——两者用"用户是否点名具体功能"这条线切得干干净净，值得所有做产品内 AI 推荐的人抄。
4. **反馈闭环工程化**。report-issue 把自然语言抱怨→结构化测试用例→GitHub 模板，Skill 质量靠真实流量回归，不靠发版祈祷。
5. **与 MCP 的缝合点极简**。整个 Skill 只在 handoff 一处引用 AWS MCP server(`aws___retrieve_skill`)，其余全部本地文件自足——符合"Skill 给脑、MCP 给手、松耦合"的总架构。

## 六、主打场景与边界

**适合**:
- "我要做个 X，该用什么数据库"——select 全流程
- "我的 Aurora 集群怎么升级/估价/迁移"——handoff 到专科
- "你刚才推荐错了"——report-issue 捕获

**明确不适合（Skill 自己写明的边界）**：不设计多服务架构（请求架构设计就明说并转交）;**选定服务后不做 schema/查询优化**("your job is done once the service is chosen");不写基础设施代码（那是专科 Skill 的活）;不替用户安装任何 Skill 或变更任何资源（同意链分离）。

## 七、风险与冷静评估

1. **指令注入面**:66 行总闸 + 20KB 决策树 + 17 张卡，全是进模型上下文的指令。从非官方渠道装 Skill 与装 MCP server 同风险级，frontmatter 的 `description` 直接决定触发行为。
2. **知识卡时效**:version 3 说明迭代快，卡内限额/价格过期概率不低（Skill 自己要求查文档兜底，正是承认这一点）。
3. **渠道偏向的边界情况**:"默认推 DSQL"同时是技术判断和商业导向，遇到卡在边界的需求（比如需要 pgvector 但又想要多区写），Skill 的答案是"无单一引擎满足，推荐 Aurora PG 主 + DSQL 作为可用性备选"——处理得还算诚实，但默认偏向本身是事实。
4. **模型遵从度是黑盒**：这套规则的效力完全取决于模型对 "STOP" 级指令的服从程度，没有任何运行时强制——它是自律，不是他律。生产场景仍需外层 Policy（如 AgentCore Gateway）兜底。
5. **token 经济学**:20KB 的 select.md 若频繁整载，上下文成本可观；好在 SKILL.md 本身设计了"按需加载 references"的两级结构。

## 八、动手路径（十分钟）

1. 任选安装：`/plugin install aws-core@claude-plugins-official`(Claude Code）或 `npx skills add aws/agent-toolkit-for-aws/skills`（通用），或配好 AWS MCP server 后问 "Search for AWS skills related to databases"（运行时检索，免安装）。
2. 问一个选型问题："我在给一个小团队做活动报名系统，该用什么数据库？"——观察是否先问澄清、第一句是否点名、给没给"会改变答案的条件"。
3. 再问一个事实问题："DynamoDB 单表多大？"——观察是否走知识卡而非训练记忆。
4. 然后纠正它："不对，你应该推荐 X。"——观察是否进入 report-issue 流程、有没有把你的话结构化成测试用例。
5. 最后试边界："帮我设计一个包含数据库、缓存、搜索的架构。"——观察它是否拒绝编排并说明边界。

---

## 参考资料

- [GitHub: agent-toolkit-for-aws skills/core-skills/aws-database（SKILL.md 总闸、子技能注册表、服务索引表，2026-10-20 逐文件抓取）](https://github.com/aws/agent-toolkit-for-aws/tree/main/skills/core-skills/aws-database)
- [GitHub: aws-database/references/select.md（20KB 选型决策树：上下文识别三表、serverless 消歧、DSQL vs Aurora PG 九行矩阵、迁移/重构路由、回答规则，2026-10-20 全文核对）](https://github.com/aws/agent-toolkit-for-aws/blob/main/skills/core-skills/aws-database/references/select.md)
- [GitHub: aws-database/references/handoff.md（交接协议：aws___retrieve_skill 直取、降级路径、同意链，2026-10-20 全文核对）](https://github.com/aws/agent-toolkit-for-aws/blob/main/skills/core-skills/aws-database/references/handoff.md)
- [GitHub: aws-database/references/report-issue.md（纠错工作流：测试用例 JSON、五类分类、渠道路由，2026-10-20 全文核对）](https://github.com/aws/agent-toolkit-for-aws/blob/main/skills/core-skills/aws-database/references/report-issue.md)
- [GitHub: aws-database/assets/（17 张服务知识卡：固定字段、llms.txt 锚点、DynamoDB SearchVectors 边界，2026-10-20 抽查核对）](https://github.com/aws/agent-toolkit-for-aws/tree/main/skills/core-skills/aws-database/assets)
- [GitHub: agent-toolkit-for-aws skills/README.md（Skill 体系说明与四种分发渠道，2026-10-20 抓取）](https://github.com/aws/agent-toolkit-for-aws/blob/main/skills/README.md)
