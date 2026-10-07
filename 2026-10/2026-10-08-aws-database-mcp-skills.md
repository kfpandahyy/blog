# 云数据库 MCP 全景调研：AWS 与阿里云

> 类型 · 定位 · 使用场景 · 功能点 · 实现原理

云厂商正在把数据库产品改造成 Agent 的原生能力。AWS 与阿里云走了两条不同的路：AWS 按引擎拆分、管理面与数据面分治；阿里云依托 DMS 统一网关一托多。本文按五个维度拆解两家布局，并给出对比与启示。

---

## 一、问题背景

传统数据库开发的痛点：开发者在 IDE、数据库客户端、文档之间反复切换，同时维护多套数据模型的心智（SQL 方言差异、关系型 vs NoSQL 建模范式差异）。MCP 的价值主张是把数据库的元数据与操作能力，以标准协议注入 AI 编码助手的工作现场，让模型"看见"真实的 schema、访问模式与运行状态，而不是凭训练记忆猜。

两家的共同方法论：**Skills 是大脑（决策与知识），MCP 是手（安全执行），Plugin 是打包单元（一条命令装齐一类负载所需的全部能力）**。

---

## 二、AWS：管理面与数据面分治

### 2.1 核心分野：两个平面，两套 Server

理解 AWS 数据库 MCP 体系的第一把钥匙：**控制面操作和数据面操作被拆成不同的 Server**，独立仓库、独立权限模型。

**管理面 MCP：RDS Management MCP Server**（`github.com/aws-rds-mcp/rds-management`，独立仓库）

- 定位：数据库资源生命周期管理——对应 AWS 控制台 / RDS API 那层能力
- 工具面：集群管理（CreateDBCluster / ModifyDBCluster / DeleteDBCluster / ChangeDBClusterStatus 启停重启 / FailoverDBCluster 故障转移）、快照与恢复（创建/删除/从快照恢复/PITR）、实例管理（Create/Modify/DeleteDBInstance）、参数组管理
- 资源模板：`aws-rds://db-cluster`、`aws-rds://db-instance`，让 Agent 先"看见"资源再操作
- 安全：`--readonly` 屏蔽一切变更；官方建议给 LLM 单独配只读 IAM 角色，与人权限分离

**数据面 MCP：以 Aurora PostgreSQL 为代表**（`awslabs/mcp` 仓库下的 `postgres-mcp-server` / `mysql-mcp-server`）

- 定位：数据操作与查询——对应 psql/mysql 客户端那层能力
- 工具面：`sql_list_tables` / `sql_get_schema`（发现）、`nl2sql`（自然语言转 SQL，LLM provider 抽象，支持 Bedrock / LiteLLM）、`sql_run_query`、`business_concepts`（业务术语映射层）、`reset_context`（schema 上下文管理）
- 执行通道：RDS Data API（HTTP，凭据走 Secrets Manager），新版支持直连连接串，自托管 PostgreSQL 也能用

**拆分的三个现实约束**：

1. **权限模型不同**：管理操作走 IAM + RDS API，数据操作走数据库账号。合在一起意味着一个 Server 同时持有两种高价值凭据
2. **风险等级不同**：DeleteDBInstance 是分钟级不可逆资损，DELETE FROM 有备份可救。分开后可对管理面默认只读、数据面放开写
3. **迭代频率不同**：数据面随方言和场景快速演进，管理面跟随 RDS API 版本走

### 2.2 数据面 MCP 矩阵

全部在 `awslabs/mcp`，按数据模型分类：

**关系型（5）**：Aurora PostgreSQL（NL2SQL）、Aurora MySQL（NL2SQL）、Aurora DSQL（分布式 SQL）、RDS Oracle（Secrets Manager 认证）、RDS MSSQL（Data API）

**NoSQL（5）**：DynamoDB（设计指导与建模）、DocumentDB（MongoDB 兼容）、Neptune（图查询 openCypher/Gremlin）、Keyspaces（Cassandra 兼容）、Timestream InfluxDB（时序）

**分析与搜索（3）**：Redshift（数仓只读查询）、S3 Tables（分析表）、OpenSearch（搜索）

**缓存（3）**：ElastiCache Valkey / Memcached、MemoryDB Valkey——偏运维诊断

覆盖逻辑：不是每服务一个 MCP，而是**每个开发者会动手操作的服务一个**；建模指导类与操作类分离。

### 2.3 三个功能范式

**执行型：Aurora MySQL/PostgreSQL——NL2SQL 管道**。发现表 → 选 schema 进上下文 → 生成 SQL → 执行 → 解释。上下文管理是显式工具（`reset_context`），说明设计者清楚 schema 挤占上下文窗口的问题。

**指导型：DynamoDB——prompt-as-tool**。2.0 把 CRUD 操作剥离给通用 AWS API MCP Server，只做设计指导：`dynamodb_data_modeling` 检索并返回建模专家 prompt；`source_db_analyzer` 从 MySQL schema + Performance Schema 提取访问模式生成 DynamoDB 设计建议。本质是**把专家知识以 prompt 形式作为工具返回**，rule-based 无需 LLM 调用。解决"关系型思维迁移 NoSQL"这个最贵的问题。

**极简型：Aurora DSQL——三工具原则**。只有 `get_schema` / `readonly_query` / `transact`，但配套 Skill 极重（DDL 迁移、OCC 重试、多租户隔离、查询计划诊断，全部在 reference 文件按需加载）。**工具面最小化，知识面最大化**。

### 2.4 安全模型（完整链路）

**数据面**：

1. 凭据不进上下文：密码存 Secrets Manager，LLM 永远看不到密钥
2. RDS Data API 作为执行通道：HTTP 接口不直连数据库端口
3. **语义级只读**：postgres-mcp-server 用 `pglast`（PostgreSQL 官方解析器）解析每条 SQL，只放行 SELECT 类语句形状；识别"伪装成 SELECT 的写操作"——`nextval()` 改序列、`pg_stat_reset()` 清统计、`pg_switch_wal()` 切日志，一律拒绝
4. **角色权限双保险**：`--privilege_check` 连接时校验角色，超级用户 / `rds_superuser` 成员 / BYPASSRLS 角色在 enforce 模式直接拒连；推荐专用最小权限角色，即使 SQL 过滤被绕过，数据库层权限仍是最后边界
5. 写前校验：DSQL `dsql_lint` 静态检查 DDL
6. Hooks 拦截：agent-plugins 的 `databases-on-aws` 插件注册 schema 验证 hook，`transact` 写后自动提示验证变更与影响行数

**管理面**：`--readonly` 整体屏蔽变更；高危操作在 AWS API MCP Server 层还有 denyList / elicitList 二次确认。

### 2.5 Skill 体系：三种形态

1. **嵌入 MCP Server 内部**：`aurora-dsql-mcp-server/skills/`，与工具同仓发行
2. **agent-plugins 插件化打包**：`databases-on-aws` = Skill + MCP（awsknowledge + aurora-dsql，后者默认禁用）+ Hooks，一条 `/plugin install` 装齐
3. **agent-toolkit-for-aws 路由中枢**：`aws-database` 核心 Skill 是入口——description 写死"STOP——不要凭训练知识回答"，意图匹配到子技能注册表（15+ 引擎）→ 知识卡片（services.json 快速事实）→ `requirements.json` artifact 交接给 service skill（避免用户重复输入）→ service skill 内部再路由（Aurora PG Skill 有 20+ 子技能，按需加载 reference）

这是**渐进式披露**的完整实现：frontmatter 触发 → 路由 → 按需读 reference → 执行。Agent 上下文里始终只有当前任务需要的知识。

### 2.6 通用执行层与托管整合

- **aws-api-mcp-server**（开源）：AWS CLI 命令封装为 MCP 工具，只读模式、沙箱、CloudTrail 审计。DynamoDB 表管理操作迁到了这里
- **AWS MCP Server（托管版，2026 GA）**：散装开源 Server 整合成单一托管端点（us-east-1 / eu-central-1），能力 = 文档检索 + API 调用 + 脚本执行 + 官方 Skills 四合一。**IAM 上下文键区分人类与 Agent 身份**，CloudTrail + CloudWatch 全审计，Server 本身免费。从"运维一堆本地 Server"切换到"接一个端点"

---

## 三、阿里云：DMS 统一网关模式

### 3.1 布局总览

阿里云的打法与 AWS 截然不同：不按引擎拆 MCP，而是把数据管理能力收敛到 **DMS（数据管理服务）** 这个已有的企业级管控面，让 DMS MCP 一个网关托住 40+ 数据源。2026 年又推出**瑶池统一 MCP**，进一步把"建实例 + 用数据 + AI 诊断"合并进一个 Server。

```
        AWS                          阿里云
数据面   每引擎一个 MCP               DMS MCP（统一网关，40+ 源）
                                    PolarDB Supabase MCP（metadata-only）
管理面   rds-management MCP          RDS OpenAPI MCP
                                    瑶池 MCP（create_instance + AI 顾问）
通用层   aws-api-mcp / 托管 GA        alibabacloud-api-mcp-server（托管，数万 OpenAPI）
AI 顾问  Skill 体系（路由+知识库）     ask_yaochi_agent 工具 / RDS AI 助手 Skill / DAS Agent
```

### 3.2 数据面旗舰：DMS MCP Server

- **仓库**：`aliyun/alibabacloud-dms-mcp-server`（2025-06 发布，"DMS 面向 AI Agent 的统一数据访问 MCP 服务"）
- **定位**：多云通用的统一数据访问网关——不只是阿里云数据库，而是任何接进 DMS 的数据源
- **数据源覆盖**：阿里云全系（RDS、PolarDB、PolarDB-X、ADB 系列、Lindorm、TableStore、MaxCompute、Hologres）+ 第三方（MySQL、MariaDB、PostgreSQL、Oracle、SQLServer、Redis、MongoDB、StarRocks、ClickHouse、SelectDB、DB2、OceanBase、Gauss、BigQuery），40+ 种
- **两种模式**：多实例模式（DBA 统一管理多环境多实例）；单库模式（`CONNECTION_STRING` 锁定一个库）
- **核心工具**：`nlsql`（内置 NL2SQL 算法：自然语言 → 匹配数据表 → 理解业务含义 → 生成并执行 SQL）、`executeScript`、`getTableDetailInfo`（schema）、`addInstance`（录入实例）；权限管控与审计日志默认随调用附带
- **安全设计**（背靠 DMS 成熟管控面）：
  - 账号密码安全托管（DMS 当凭证管家，AK/SK 只做认证）
  - 内网访问，数据不出域
  - 细粒度权限：实例 / 库 / 表 / 字段 / 行级
  - 高危 SQL 规则引擎实时拦截（如无条件 DELETE、全表扫描）
  - 全量 SQL 审计日志
  - 网络层仅绑定 127.0.0.1，不接受远程连接
- **托管形态**：DMS 控制台内可直接开通 MCP 服务（公测期免费），无需本地部署

### 3.3 管理面：RDS OpenAPI MCP + 瑶池统一 MCP

**RDS OpenAPI MCP Server**（`aliyun/alibabacloud-rds-openapi-mcp-server`）：

- OpenAPI 工具集：实例创建、查询、变配（如 `create_db_instance` / `describe_db_instances` / `modify_db_instance_spec`），按 toolset 组织（`rds`、`rds_mssql_custom` 等），启动时可裁剪
- SQL 工具：自动创建只读账号执行查询，完毕即删
- 附带 **RDS AI 助手 Claude Skill**（skill/ 目录）：SQL 优化、实例运维、故障排查
- 传输：SSE / stdio

**瑶池数据库 MCP Server**（`aliyun/alibabacloud-yaochi-db-mcp-server`，2026）：

- 定位一句话：**一个 MCP 统一管理阿里云全系数据库**，并内置 AI 顾问
- 引擎：RDS MySQL、PolarDB MySQL、MongoDB、Tair（Redis），向瑶池全系扩展
- 工具面把管理面、数据面、AI 顾问装进同一个 Server：
  - `create_instance` / `list_instances`（管理面）
  - `execute_instance_sql`（**临时账号模式**：自动开通公网 + 白名单 + 建库 + 执行，无需密码）、`execute_mysql` / `execute_mongo` / `execute_redis`（直连）
  - `ask_yaochi_agent`：瑶池数据库 Agent——AI 智能顾问（知识问答、性能诊断、最佳实践），以工具形式暴露
  - 还有 DMS 桥接：`search_database` / `execute_sql` / `register_to_dms`
- 写操作通过环境变量显式开启：`YAOCHI_ENABLE_WRITE_SQL` / `YAOCHI_ENABLE_DDL_SQL`
- 核心场景：AI 写完代码 → 自动建库 → 建表 → 执行 SQL 验证 → 调 Agent 做性能诊断，全程不出 IDE

### 3.4 PolarDB Supabase MCP Server

- **仓库**：`ApsaraDB/PolarDB-Supabase-MCP-Server`（TypeScript / pnpm）
- 定位：为 AI 原生 IDE（Qoder 等）提供 **metadata-only** 通道——安全暴露表、列、类型、约束，**不暴露业务数据**
- 这是"vibe coding"场景的专用设计：模型只需要 schema 来生成代码，不需要碰数据

### 3.5 通用层：阿里云 OpenAPI MCP Server

`aliyun/alibabacloud-api-mcp-server`：官方托管的远程 MCP，覆盖**数万个阿里云 OpenAPI**，无需本地部署。特色能力：OpenAPI 描述为 AI 调优（精简非必填参数）、Terraform as Tools（HCL 代码即工具，变量自动转参数，确定性编排）、多账号角色扮演、自定义 OAuth（最长一年免登录）、MCP Proxy 内置遥测可视化。按产品维度也提供本地 stdio 独立 Server。

### 3.6 Skill 与 Agent 配套

相比 AWS 的三层 Skill 体系，阿里云的 Skill 故事更薄，**Agent 以工具形式内嵌**：

- RDS AI 助手 Skill（Claude Skill 形态，挂 RDS OpenAPI MCP）
- `ask_yaochi_agent`：瑶池 Agent 直接做成 MCP 工具（知识问答 / 智能诊断 / 最佳实践，融合官方文档知识库与专家经验）
- DAS Agent：数据库自治运维大脑（融合 10 万+ 工单与专家经验，问题发现 → 诊断 → 优化全链路自治）
- DMS Data Copilot / Data Agent：数据管理智能助手（元数据 + 问数知识库）
- 通义灵码 IDE 侧集成：DMS MCP + 灵码的组合是官方主推的开发提效路径

### 3.7 安全模型

DMS MCP 的安全栈与 AWS 思路相同但落点不同——AWS 靠协议与解析器（Data API / pglast），阿里云靠已有管控面产品（DMS）：

- 凭证托管：DMS 安全托管实例账号，KMS 凭据支持
- 网络：内网访问（数据不出域）+ 127.0.0.1 本地绑定
- 权限：实例 / 库 / 表 / 字段 / 行级细粒度管控
- 执行：高危 SQL 规则引擎实时识别拦截，DMS 安全托管模式内置 SQL 审核与审批流
- 审计：全量 SQL 操作日志，合规可追溯
- 瑶池 MCP 的写操作默认关闭，环境变量显式开启

---

## 四、AWS vs 阿里云：五个维度的对比

| 维度 | AWS | 阿里云 |
|---|---|---|
| 架构哲学 | 每引擎窄 MCP，管理/数据分治，通用 API 兜底 | DMS 统一网关一托多，瑶池走向"一个 MCP 管全系" |
| 数据面 | 12+ 个按引擎拆分（NL2SQL 各自为政） | DMS MCP 一个网关 40+ 源，NL2SQL 统一算法 |
| 管理面 | 独立 rds-management 仓库，实例/集群/快照/参数组 | RDS OpenAPI MCP（toolset 裁剪）+ 瑶池 create_instance |
| AI 知识注入 | Skill 三层体系（渐进披露、路由、artifact 交接），模型"不许凭记忆回答" | Agent 工具化（ask_yaochi_agent）、Skill 形态仅一处，依托 DAS/DMS 既有 AI 产品 |
| 安全抓手 | Data API 通道、pglast 语义只读、privilege_check、IAM 上下文键 | DMS 管控面：行级权限、高危 SQL 规则引擎、审计、内网、127.0.0.1 |
| 托管形态 | AWS MCP Server GA（单一端点四合一） | DMS 控制台内开通 MCP（公测免费）+ 托管 OpenAPI MCP |
| 数据源开放性 | 只覆盖 AWS 自家引擎 | 多云通用（含 OceanBase、Gauss、BigQuery 等对手产品） |

**深层差异**：AWS 的优势在"每个引擎做深"——语义级只读、业务概念层、建模指导都是引擎级精细设计；阿里云的优势在"网关做宽"——DMS 十年管控面积累（权限、审计、审批）直接复用，40+ 源统一接入是 AWS 没有的能力。AWS 像"每个数据库配一个专家助手"，阿里云像"一个 DBA 总管所有库"。

**收敛趋势**：两家都在走向同一终点——托管 MCP 端点 + AI 顾问工具化 + 人/Agent 身份分离 + 全链路审计。差异只在路径：AWS 从开源散装到托管整合，阿里云从管控面产品长出来。

---

## 五、使用场景（合并两家）

1. **Schema 驱动的特性开发**：AI 读取实时 schema 理解表关系，生成 CRUD 代码（AWS Aurora MCP / 阿里云 DMS MCP + 灵码）
2. **数据探索与业务洞察**：分钟级 dashboard，DMS NL2SQL + 问数知识库面向非技术人员
3. **测试代码生成**：基于 live schema 和访问模式生成针对性测试
4. **监控与排障**：缓存内存、复制状态、慢查询自然语言查询；瑶池 Agent / DAS Agent 自动诊断
5. **异构迁移**：MySQL → DynamoDB 建模转换（AWS）；跨源统一访问（阿里云 DMS）
6. **ChatBI / 数据民主化**：Redshift 只读查询（AWS）；DMS Data Agent 分析报告（阿里云）
7. **自然语言运维**：实例变配、快照、故障转移（AWS rds-management）；建实例 + 诊断一体（阿里云瑶池）
8. **Vibe coding 配套**：AI 写代码后自动建库建表验证数据（阿里云瑶池核心场景；PolarDB Supabase MCP 供 metadata）

---

## 六、观察与启示

**1. 管理面与数据面分治是 Agent 时代的刚需。** 权限模型、风险等级、迭代节奏三者都不同。做数据库 MCP 的产品，第一步就该想清楚这层拆分。

**2. 两种架构范式都有生存空间，取决于你手里有什么。** 有成熟管控面产品（DMS 模式）→ 统一网关；引擎各自为战且深度差异大（AWS 模式）→ 按引擎拆分 + 通用兜底。手里没有管控面的厂商，更可能走 AWS 路线。

**3. 工具面做窄，知识面做宽。** DSQL 三工具 + 重 Skill，DynamoDB 只做设计指导，PolarDB Supabase 只给 metadata。Agent 时代的工具设计是"最小安全操作面 + 最大决策知识"。

**4. 安全是分层工程。** AWS：凭据隔离 → 执行通道隔离 → 语义级只读（pglast 识别伪装的 SELECT）→ 角色权限双保险 → 写前 lint → 写后 hook → CloudTrail。阿里云：凭证托管 → 内网 + 本地绑定 → 行级权限 → 高危 SQL 规则引擎 → 审批流 → 审计。每一层由不同组件承担。

**5. AI 顾问的两种交付形态。** AWS 把知识做成 Skill（可读 Markdown，渐进披露，路由）；阿里云把顾问做成工具（ask_yaochi_agent / DAS Agent）。前者轻、可组合、依赖模型能力；后者重、确定性高、自带执行闭环。两者可能 converge 到"Skill 定义流程 + 工具执行动作"。

**6. "不要凭训练知识回答" vs "问数知识库"。** 两家殊途同归：模型记忆不可信，检索与受控执行才可信。

**7. 对竞争的意味：** 数据库厂商的下一个竞争维度，是元数据接口、文档体系、Skill 包、管理面 MCP 对 Agent 的友好度。AWS 和阿里云已经把"数据库产品 = Agent 原生能力"当成既定战略——其他厂商跟不跟、怎么跟，是接下来一年最值得看的事。

---

## 参考资料

**AWS**

1. AWS Database Blog: Supercharging AWS database development with AWS MCP servers (2025-06) — https://aws.amazon.com/blogs/database/supercharging-aws-database-development-with-aws-mcp-servers/
2. GitHub: awslabs/mcp — https://github.com/awslabs/mcp
3. GitHub: aws-rds-mcp/rds-management（管理面 MCP）— https://github.com/aws-rds-mcp/rds-management
4. GitHub: awslabs/agent-plugins — https://github.com/awslabs/agent-plugins
5. GitHub: aws/agent-toolkit-for-aws — https://github.com/aws/agent-toolkit-for-aws
6. awslabs.postgres-mcp-server PyPI（pglast 语义只读与 privilege_check）— https://pypi.org/project/awslabs.postgres-mcp-server/
7. serverworks blog: AWS MCP Server GA 移行记 (2026-05) — https://blog.serverworks.co.jp/aws-mcp-server-ga-2026
8. GitHub: awslabs/mcp — aws-api-mcp-server — https://github.com/awslabs/mcp/tree/main/src/aws-api-mcp-server

**阿里云**

9. GitHub: aliyun/alibabacloud-dms-mcp-server（DMS 统一数据访问 MCP）— https://github.com/aliyun/alibabacloud-dms-mcp-server
10. 阿里云文档: 使用 DMS MCP 让大模型安全访问数据库 — https://help.aliyun.com/zh/dms/use-cases/deploy-dms-mcp
11. 阿里云开发者社区: 告别切屏｜DMS MCP + 通义灵码 30 分钟搞定电商秒杀开发 (2025-06) — https://developer.aliyun.com/article/1666619
12. GitHub: aliyun/alibabacloud-rds-openapi-mcp-server（RDS OpenAPI MCP + RDS AI 助手 Skill）— https://github.com/aliyun/alibabacloud-rds-openapi-mcp-server
13. GitHub: aliyun/alibabacloud-yaochi-db-mcp-server（瑶池统一 MCP）— https://github.com/aliyun/alibabacloud-yaochi-db-mcp-server
14. GitHub: ApsaraDB/PolarDB-Supabase-MCP-Server（metadata-only）— https://github.com/ApsaraDB/PolarDB-Supabase-MCP-Server
15. 阿里云文档: PolarDB Supabase 助力 AI 原生 IDE 完成 VibeCoding — https://help.aliyun.com/zh/polardb/polardb-for-postgresql/polardb-supabase-ai-ide-vibecoding
16. GitHub: aliyun/alibabacloud-api-mcp-server（托管 OpenAPI MCP）— https://github.com/aliyun/alibabacloud-api-mcp-server
17. 阿里云: 瑶池数据库 Data+AI 开放日（DAS Agent / DMS Data Copilot / DMS MCP 发布）— https://www.aliyun.com/activity/database/data4ai-openday
