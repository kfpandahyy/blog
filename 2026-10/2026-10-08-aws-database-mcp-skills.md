# AWS 数据库 MCP 全景调研

> **TL;DR**：AWS 把数据库 MCP 拆成管理面（RDS Management MCP，独立仓库、默认只读）和数据面（12+ 个按引擎拆分的 Server，NL2SQL 为主力）两个平面分治，2026 年推出四合一托管端点（AWS MCP Server GA，免费）。选型一句话：查数用 postgres/mysql server，运维用 rds-management（`--readonly` 起步），全托管省事直接接官方端点。
> **数据截至 2026-10。** 该领域迭代极快，厂商能力以官方渠道最新公告为准。

---

AWS 的客户一直在问同一个问题：怎么把现有数据库系统接进 AI 编码助手，让模型"看见"真实的 schema 而不是凭训练记忆猜。AWS 的答案是到 2026 年已形成完整栈：8+ 数据库专用 MCP Server、独立的管理面 Server、插件化的 Skill 体系、以及一个托管的 GA 端点。

本文的路线：先给格局地图（管理面 / 数据面 / 通用层 / 打包托管四层），再逐层同构拆解，最后给选型决策树和风险提示。

---

## 一、调研口径

- **信息源**：AWS 官方博客与文档（一手）、GitHub 仓库 awslabs/mcp / aws-rds-mcp / awslabs/agent-plugins / aws/agent-toolkit-for-aws（一手）、PyPI 包说明（一手）、serverworks 等第三方实测博客（二手）。
- **检索语言**：中英文。
- **免责声明**：厂商能力以快照为准，GA 边界和区域可用性变动最快。

---

## 二、格局总览：四层架构

```
┌─ 打包托管层 ──────────────────────────────────────┐
│ agent-plugins（databases-on-aws 插件）              │
│ agent-toolkit-for-aws（aws-database 路由 Skill）    │
│ AWS MCP Server 托管端点（2026 GA，四合一，免费）      │
├─ 通用执行层 ──────────────────────────────────────┤
│ aws-api-mcp-server（CLI→工具，只读/沙箱/CloudTrail）│
├─ 管理面 ──────────────────────────────────────────┤
│ RDS Management MCP（独立仓库，--readonly，IAM 角色）│
├─ 数据面（awslabs/mcp，12+ 个）─────────────────────┤
│ 关系型：Aurora PG / Aurora MySQL / DSQL / RDS      │
│          Oracle / RDS MSSQL                        │
│ NoSQL：DynamoDB / DocumentDB / Neptune / Keyspaces │
│        / Timestream InfluxDB                       │
│ 分析搜索：Redshift / S3 Tables / OpenSearch        │
│ 缓存：ElastiCache Valkey/Memcached、MemoryDB       │
└───────────────────────────────────────────────────┘
```

理解 AWS 体系的第一把钥匙：**控制面操作和数据面操作被拆成不同的 Server**，独立仓库、独立权限模型。这是本文的核心叙事，逐层拆解时反复用到。

共同方法论一句话：**Skills 是大脑（决策与知识），MCP 是手（安全执行），Plugin 是打包单元（一条命令装齐一类负载所需的全部能力）**。

---

## 三、逐层拆解

### 3.1 数据面双雄：postgres-mcp-server / mysql-mcp-server

- **一句话定位**：关系型数据面的主力，NL2SQL 管道 + 业务概念层，自托管数据库也能用。
- **盘点**：`sql_list_tables` / `sql_get_schema`（发现）→ `nl2sql`（自然语言转 SQL，LLM provider 抽象，支持 Bedrock / LiteLLM）→ `sql_run_query`（执行）→ `business_concepts`（业务术语映射层）→ `reset_context`（schema 上下文管理）。
- **解剖**：执行通道默认走 RDS Data API（HTTP，凭据存 Secrets Manager），新版支持直连连接串——凭据不进上下文，模型永远看不到密码。`business_concepts` 是差异化设计：让模型先学组织的业务术语再生成 SQL。`reset_context` 做成显式工具，说明设计者清楚 schema 挤占上下文窗口的问题。
- **安全模型**：语义级只读——用 `pglast`（PostgreSQL 官方解析器）解析每条 SQL，只放行 SELECT 类语句形状，能识别"伪装成 SELECT 的写操作"（`nextval()` 改序列、`pg_stat_reset()` 清统计、`pg_switch_wal()` 切日志，一律拒绝）；`--privilege_check` 连接时校验角色，超级用户 / `rds_superuser` 成员 / BYPASSRLS 在 enforce 模式直接拒连；推荐专用最小权限角色做数据库层最后边界。
- **成熟度**：开源活跃，PyPI 分发；NL2SQL 依赖外部 LLM provider，生产需自行评估生成质量。

### 3.2 数据面极简范式：Aurora DSQL MCP

- **一句话定位**：三工具原则的代表——工具面最小化，知识面最大化。
- **盘点**：只有 `get_schema` / `readonly_query` / `transact` 三个工具。
- **解剖**：配套 Skill 极重——DDL 迁移、乐观并发控制（OCC）重试、多租户隔离、查询计划诊断，全部放在 reference 文件里按需加载。这是渐进式披露的完整实现：frontmatter 触发 → 按需读 reference → 执行。写操作前有 `dsql_lint` 静态检查 DDL。
- **安全模型**：只读查询与写入显式分离（`readonly_query` vs `transact`）；写前 lint + agent-plugins 的 schema 验证 hook 双保险。
- **成熟度**：随 DSQL 同步演进，范式意义大于规模意义。

### 3.3 数据面指导范式：DynamoDB MCP

- **一句话定位**：不做 CRUD，只做设计指导——prompt-as-tool 的代表。
- **盘点**：`dynamodb_data_modeling`（检索并返回建模专家 prompt）；`source_db_analyzer`（从 MySQL schema + Performance Schema 提取访问模式，生成 DynamoDB 设计建议）。2.0 起表管理类 CRUD 剥离给通用 AWS API MCP Server。
- **解剖**：本质是**把专家知识以 prompt 形式作为工具返回**，rule-based 无需 LLM 调用。解决的是"关系型思维迁移 NoSQL"这个最贵的问题——DynamoDB 建错表的成本远高于查错数。
- **安全模型**：不直接执行数据操作，攻击面天然小；建模建议输出依赖模型判断，落地仍需人工评审。
- **成熟度**：2.0 架构收敛后定位清晰，是三种范式里设计最克制的一个。

### 3.4 数据面其余专用 Server

合并速览（同构信息较少，表格承载）：

| Server | 一句话定位 | 特色 |
|---|---|---|
| RDS Oracle | 企业存量库接入 | Secrets Manager 认证 |
| RDS MSSQL | 微软生态接入 | Data API 通道 |
| DocumentDB | MongoDB 兼容 | 文档模型操作 |
| Neptune | 图查询 | openCypher / Gremlin |
| Keyspaces | Cassandra 兼容 | 宽列操作 |
| Timestream InfluxDB | 时序 | InfluxDB 协议 |
| Redshift | 数仓 | 只读查询 |
| S3 Tables | 分析表 | 湖仓新形态 |
| OpenSearch | 搜索 | 检索语义 |
| ElastiCache / MemoryDB | 缓存运维 | 偏诊断（内存、复制状态） |

覆盖逻辑值得注意：不是每服务一个 MCP，而是**每个开发者会动手操作的服务一个**；建模指导类与操作类分离。

### 3.5 管理面：RDS Management MCP

- **一句话定位**：数据库资源生命周期管理——对应 AWS 控制台 / RDS API 那层能力，与数据面彻底分治。
- **盘点**：集群管理（CreateDBCluster / ModifyDBCluster / DeleteDBCluster / ChangeDBClusterStatus 启停重启 / FailoverDBCluster 故障转移）、快照与恢复（创建 / 删除 / 从快照恢复 / PITR）、实例管理、参数组管理；资源模板 `aws-rds://db-cluster`、`aws-rds://db-instance` 让 Agent 先"看见"资源再操作。
- **解剖**：独立仓库 `aws-rds-mcp/rds-management`（不在 awslabs/mcp 里），迭代节奏跟随 RDS API 版本，与数据面的快速演进解耦。
- **安全模型**：`--readonly` 整体屏蔽一切变更（起步默认值）；官方建议给 LLM 单独配只读 IAM 角色，与人权限分离；高危操作在 AWS API MCP Server 层还有 denyList / elicitList 二次确认。
- **成熟度**：面向运维场景，是自然语言运维（"把集群停了省点钱"）的载体。

**分治的三个现实约束**（本节是全文核心判断）：

1. **权限模型不同**：管理操作走 IAM + RDS API，数据操作走数据库账号。合并 = 一个 Server 同时持有两种高价值凭据。
2. **风险等级不同**：DeleteDBInstance 分钟级不可逆资损，DELETE FROM 有备份可救。分开后可对管理面默认只读、数据面放开写。
3. **迭代频率不同**：数据面随方言和场景快速演进，管理面跟随 API 版本走。

### 3.6 通用执行层：aws-api-mcp-server

- **一句话定位**：AWS CLI 命令封装为 MCP 工具，兜底一切专用 Server 没覆盖的操作。
- **盘点**：只读模式、沙箱执行、CloudTrail 审计；DynamoDB 的表管理操作已迁移至此。
- **安全模型**：只读默认 + 审计全覆盖，是四层架构里的"通用但受限"通道。
- **成熟度**：awslabs/mcp 内最通用的 Server，也是 DynamoDB 2.0 收敛的受益者。

### 3.7 打包托管层：从散装到 GA

- **一句话定位**：把开源散装 Server 和 Skill 整合成"装得上、管得住"的交付形态。
- **盘点（三种形态）**：
  1. **嵌入 MCP Server 内部**：`aurora-dsql-mcp-server/skills/`，与工具同仓发行；
  2. **agent-plugins 插件化**：`databases-on-aws` = Skill + MCP + Hooks，一条 `/plugin install` 装齐；
  3. **agent-toolkit-for-aws 路由中枢**：`aws-database` Skill 是入口——description 写死"STOP——不要凭训练知识回答"，意图匹配到 15+ 引擎的子技能注册表 → 知识卡片（services.json 快速事实）→ `requirements.json` artifact 交接给 service skill（避免用户重复输入）→ Aurora PG Skill 内部还有 20+ 子技能按需加载。
- **解剖**：路由 Skill 是渐进式披露的工程化样板——Agent 上下文里始终只有当前任务需要的知识。这也是本博客《Agent Skill 调研》里"描述即路由"原则的生产级实现。
- **安全模型**：托管版 **AWS MCP Server（2026 GA）**——单一端点（us-east-1 / eu-central-1），能力 = 文档检索 + API 调用 + 脚本执行 + 官方 Skills 四合一；**IAM 上下文键区分人类与 Agent 身份**，CloudTrail + CloudWatch 全审计，Server 本身免费。
- **成熟度**：2026 年 GA 是标志性节点——从"运维一堆本地 Server"切换到"接一个端点"。

---

## 四、横向对比与选型

### 四层对比

| 维度 | 数据面（PG/MySQL） | 数据面（DSQL/DynamoDB） | 管理面 RDS | 托管 GA 端点 |
|---|---|---|---|---|
| 干什么 | NL2SQL 查数 | 写事务 / 建模指导 | 实例集群生命周期 | 四合一托管 |
| 写操作 | 受控 | lint + hook / 无 | 默认禁（--readonly） | IAM 控制 |
| 安全抓手 | pglast + privilege_check | 工具面最小化 | IAM 角色隔离 | 身份键 + 全审计 |
| 适合谁 | 开发者查数 | DSQL 用户 / NoSQL 建模 | DBA / 运维 | 全托管团队 |

### 选型决策树

```
你要干什么？
├─ 自然语言查数 / 生成 SQL
│   └─→ postgres-mcp-server / mysql-mcp-server（生产加 --privilege_check）
├─ 建模范式迁移（MySQL → DynamoDB）
│   └─→ dynamodb-mcp 的 source_db_analyzer
├─ 运维操作（启停 / 快照 / 参数 / 故障转移）
│   └─→ rds-management MCP，永远先 --readonly
├─ 以上都要，且不想运维本地 Server
│   └─→ AWS MCP Server 托管端点（确认区域可用性）
└─ 兜底（专用 Server 没覆盖的服务操作）
    └─→ aws-api-mcp-server（保持只读默认）
```

---

## 五、风险与未决问题

1. **区域可用性**：托管 GA 端点仅 us-east-1 / eu-central-1（截至 2026-10），中国区域不可用，跨境团队需评估。
2. **开源散装的运维税**：自建 Server 的版本跟进、IAM 配置、密钥管理都是隐性成本——GA 端点能省多少，取决于你是否愿意接受它的能力边界（四合一 ≠ 数据面全部精细能力）。
3. **NL2SQL 的质量责任**：生成 SQL 的正确性最终由你承担，pglast 只保证"只读形状"，不保证语义正确。生产环境保留人工确认环节。
4. **迭代速度带来的 breaking change**：awslabs/mcp 高频迭代（DynamoDB 2.0 架构收敛就是例子），锁定版本并读 changelog 是必修课。

---

## 六、观察与启示

1. **管理面与数据面分治是 Agent 时代的刚需**——权限模型、风险等级、迭代节奏三者不同。做数据库 MCP 的产品，第一步就该想清楚这层拆分。
2. **工具面做窄，知识面做宽**——DSQL 三工具 + 重 Skill、DynamoDB 只做设计指导。Agent 时代的工具设计是"最小安全操作面 + 最大决策知识"。
3. **安全是分层工程**——凭据隔离 → 执行通道隔离 → 语义级只读 → 角色权限双保险 → 写前 lint → 写后 hook → CloudTrail 审计。每层由不同组件承担，没有银弹。
4. **"不要凭训练知识回答"是工程纪律**——路由 Skill 把这句话写进 description，本质是承认模型记忆不可信，检索与受控执行才可信。
5. **托管化是终局**——从开源散装到单一 GA 端点，AWS 用一年时间走完了"协议 → 生态 → 托管服务"的完整路径。数据库厂商的下一个竞争维度，是元数据接口、Skill 包、管理面 MCP 对 Agent 的友好度。

---

## 七、结论：给不同读者的一句话

- **开发者**：装 postgres-mcp-server 接 Aurora，权限检查参数一个都别省；运维需求先 `--readonly` 跑一周再谈写操作。
- **架构师**：评估"自建散装 vs 托管端点"时，把 IAM 配置复杂度和区域可用性放进决策表，别只比功能清单。
- **产品经理**：AWS 的路径（开源协议 → 生态 → 托管服务 → 身份分离审计）是云能力 Agent 化的标准剧本，抄结构比抄功能重要。

---

## 附录：数据面 Server 全清单（截至 2026-10）

| 类别 | Server | 核心能力 |
|---|---|---|
| 关系型 | Aurora PostgreSQL | NL2SQL + 业务概念层 |
| 关系型 | Aurora MySQL | NL2SQL |
| 关系型 | Aurora DSQL | 三工具 + 重 Skill |
| 关系型 | RDS Oracle / MSSQL | 存量企业库接入 |
| NoSQL | DynamoDB | 设计指导（prompt-as-tool） |
| NoSQL | DocumentDB / Neptune / Keyspaces / Timestream | 各模型原生操作 |
| 分析 | Redshift / S3 Tables | 只读分析查询 |
| 搜索 | OpenSearch | 检索 |
| 缓存 | ElastiCache / MemoryDB | 运维诊断 |
| 管理面 | RDS Management | 生命周期管理（独立仓库） |
| 通用 | aws-api-mcp-server | CLI 兜底 |
| 托管 | AWS MCP Server | 四合一 GA 端点 |

---

## 参考资料

### 一手（官方）

1. [AWS Database Blog: Supercharging AWS database development with AWS MCP servers（2025-06）](https://aws.amazon.com/blogs/database/supercharging-aws-database-development-with-aws-mcp-servers/)
2. [GitHub: awslabs/mcp](https://github.com/awslabs/mcp)
3. [GitHub: aws-rds-mcp/rds-management（管理面 MCP）](https://github.com/aws-rds-mcp/rds-management)
4. [GitHub: awslabs/agent-plugins](https://github.com/awslabs/agent-plugins)
5. [GitHub: aws/agent-toolkit-for-aws](https://github.com/aws/agent-toolkit-for-aws)
6. [awslabs.postgres-mcp-server PyPI（pglast 语义只读、privilege_check）](https://pypi.org/project/awslabs.postgres-mcp-server/)
7. [GitHub: aws-api-mcp-server](https://github.com/awslabs/mcp/tree/main/src/aws-api-mcp-server)

### 二手（第三方实测）

8. [serverworks blog: AWS MCP Server GA 移行记（2026-05）](https://blog.serverworks.co.jp/aws-mcp-server-ga-2026)
