# AWS 数据库 MCP 全景:工具、Skill 与托管网关的三层版图

> **TL;DR**:AWS 的数据库 MCP 版图分两层--开源侧是 `awslabs/mcp` 仓库(9.8k stars,2025-03 创建,Apache-2.0)里 20+ 个仅支持 stdio 的本地 server,覆盖 DynamoDB、Aurora(PG/MySQL/DSQL)、Redshift、DocumentDB、Neptune、Keyspaces、Timestream、Oracle/MSSQL、Valkey/Memcached、S3 Tables 及 Glue/EMR/Athena 数据管道,分发方式是 uvx + FastMCP,认证走 RDS Data API 或直连 + Secrets Manager;**控制面(RDS 管理)另有主线:RDS 服务团队在独立组织 `aws-rds-mcp` 孵化了 rds-monitoring / rds-management / rds-control-plane 三个 MCP server(RFC #617),但截至 2026-10 尚未并入 awslabs/mcp,也未上 PyPI,只能从源码运行**;托管侧是 Bedrock AgentCore 的 Gateway(把 OpenAPI/Smithy/Lambda 转成 MCP 工具的双向认证网关)+ Registry + Policy,以及 **2026-05 已 GA** 的官方托管 AWS MCP Server(IAM context keys、零凭证暴露、CloudTrail 全审计)。**两个 MCP 仓库本体都不带 Skill--官方配套 Skill 住在第三个仓库 `aws/agent-toolkit-for-aws`(已 GA):一个总路由 Skill(`aws-database`,v3)+ 15 个数据库专科 Skill(rds-oss、amazon-aurora-postgresql 等),干的是'决策与护栏',MCP 干的是'执行',脑手分离、松耦合互不互链**。选型一句话:本地开发接开源 server,RDS 控制面用 aws-rds-mcp(接受"未 GA 团队仓库"的前提),生产共享工具走 Gateway 托管,数据面操作务必开 `--readonly` 或套 Policy,选型/治理能力从 Agent Toolkit 的 Skill 层拿。
> **数据截至 2026-10-20。** 开源侧以 `awslabs/mcp` 仓库 README 与各 server src README 为准,aws-rds-mcp 侧以组织内三个仓库与 awslabs/mcp RFC #617 为准,托管侧以 AWS 官方公告与文档为准,Skill 侧以 `aws/agent-toolkit-for-aws` 仓库的 SKILL.md 原文为准(2026-10-20 逐文件核对)。

---

你的二分法是对的,而且比表面更深一层:AWS 在数据库 MCP 这件事上同时跑着**两条产品路线**--一条把 20 多个数据库的 MCP server 当开源软件发(本地 stdio、自己跑、自己管凭证),一条把 MCP 当托管云服务卖(AgentCore Gateway 托管端点、按量计费、IAM 集成)。前者解决"我的开发机上 agent 能不能用数据库",后者解决"全公司的 agent 怎么安全地共享同一批工具"。两条路线的技术取舍几乎处处相反。

但还有第三块拼图不在任何一条路线里:**RDS 服务团队自己的 GitHub 组织 `aws-rds-mcp`**--控制面 MCP(建删改实例/集群/参数组/快照 + 监控诊断)的真正主角在这里孵化,不在 awslabs/mcp 主仓。搞清楚这个组织的来龙去脉,才能解释为什么"AWS 官方数据库 MCP"看起来控制面缺位。

第四块拼图是**配套 Skill 层**:MCP server 只给模型"手"(工具),不给"脑"(决策知识与护栏)--后者由 `aws/agent-toolkit-for-aws` 仓库的 SKILL.md 体系提供,与 MCP 仓库松耦合、互不互链,但语义上严丝合缝(2.6 专章)。

本文的路线:先盘点开源 server 清单与架构,再专章解剖 aws-rds-mcp 组织,然后拆 Skill 层与托管 AgentCore 栈,接着对比路线,最后给选型和动手路径。

一句话心智模型:**开源路线是"把数据库的钥匙交给本机的 agent",托管路线是"把数据库放在 agent 够得着但有门卫的柜台后面",而 aws-rds-mcp 是"RDS 团队自己打造的仪表盘,还没搬进 4S 店"。**

---

## 一、定义与边界

- **调研对象**:AWS 官方(awslabs / AWS 服务团队)发布的数据库相关 MCP server 与托管 MCP 能力;不含社区第三方 server,不含非数据库类(仅在与数据库场景交叉时提及)。
- **"管理服务 vs 数据服务"的口径**:管理服务 = 控制面操作(建删实例/集群、改配置、参数组、快照、扩节点、查监控);数据服务 = 数据面操作(查询、读写、执行 SQL)。这个二分贯穿全文,因为两条路线在两者上的取舍不同。
- **信息源**:一手(GitHub 仓库、AWS 官方文档、AWS 公告)为主;所有数字带日期。注意 awslabs/mcp 根 README 的表格偶有与其 src 下各 server README 不一致之处,本文以 src README 为准并逐条标注。

## 二、开源路线:awslabs/mcp 仓库解剖

### 2.1 基本面

- 仓库 `awslabs/mcp`,2025-03-21 创建,Apache-2.0,截至 2026-10-19 约 9.8k stars / 1.8k forks,最近 push 2026-10-16,仍在活跃维护。
- **传输:仅 stdio**。2025-05-26 起全部移除 SSE 支持,正在向 Streamable HTTP 迁移--与 MCP 协议本身的演进节奏一致(协议 2026-07 版核心已转向无状态、每请求自携带协商信息)。
- **技术栈**:Python + FastMCP 框架,通过 `uvx` 一键运行,例如 `uvx awslabs.mysql-mcp-server@latest`。Cursor、VS Code、Kiro、Claude Code、Windsurf、Amazon Q CLI 都有对应的一键安装配置;也支持 Docker 运行。
- **官方已发出迁移信号**:README 顶部公告称 "Agent Toolkit for AWS 已上线,是 MCP servers、plugins、skills 的继任者"(2026-05 发布)--增加了 IAM 条件键(区分 agent 操作与人类操作)、CloudWatch/CloudTrail 可见性、经过准确性评估的 skills。awslabs/mcp 继续维护但新重心在 Agent Toolkit。**这个"skills"不是形容词,是一个独立的资产层--见 2.6 专章**。
- 仓库还在持续长出新 server(Amazon Translate、AppSync、Network、SAP 管理、SageMaker Spark 排障/升级等),数据库主清单以 README "SQL & NoSQL Databases" 一节为准。

### 2.2 数据库 server 清单(按"管理服务/数据服务"重排)

**数据服务(数据面)**:

| Server | 数据访问方式 | 备注 |
|---|---|---|
| Aurora PostgreSQL | RDS Data API 或 pgwire 直连 | 支持多 endpoint,LLM 提示切换连接方式 |
| Aurora MySQL | RDS Data API 或 mysqlwire 直连 | 也支持 RDS MySQL 与自建 MySQL/MariaDB |
| Aurora DSQL | 直连 | 分布式 SQL,PG 兼容 |
| RDS Oracle | Secrets Manager 取密 + python-oracledb thin 直连 | 免 Oracle Instant Client |
| RDS MSSQL | Secrets Manager 取密直连 | **根 README 表格写"via RDS Data API"是错的,以其 src README 为准** |
| DynamoDB | AWS API | 两个版本:设计指导/数据建模版 + 完整表管理版 |
| Redshift | SQL 查询 | 数仓操作与分析查询 |
| DocumentDB | MongoDB 兼容协议 | 文档操作 |
| Neptune | openCypher / Gremlin | 图查询 |
| Keyspaces | Cassandra 兼容 | 宽列操作 |
| Timestream (InfluxDB) | InfluxDB 兼容 | 时序 |
| S3 Tables | AWS API | 面向分析的表存储 |
| ElastiCache/MemoryDB for Valkey | Valkey 协议 | 高级数据结构 |
| ElastiCache for Memcached | Memcached 协议 | 缓存读写 |
| AWS Data Processing | Glue / EMR-EC2 / Athena | 管道可见性 + 执行 |
| IoT SiteWise | SiteWise API | 工业 IoT 数据 |
| Bedrock Knowledge Bases | 检索 API | 带引用的知识查询 |
| Kendra / Q Index | 检索 API | 企业搜索 |
| Lambda Tool | 执行 Lambda | 把私有资源访问包装成工具 |

**两个边界变化(2026-10 核实)**:
- **OpenSearch 已移出 awslabs/mcp**:`src/opensearch-mcp-server` 目录不存在(404),根 README 仅保留一条外链,指向社区项目 `opensearch-project/opensearch-mcp-server-py`。它不再是 AWS 官方维护的 server。
- **MSK 没有 MCP server**:当前 README 与 src 均无 MSK 条目(只有 Amazon MQ)。早前部分资料里的 MSK 说法不要沿用。

**管理服务(控制面)**:ElastiCache(完整控制面操作)、S3 Tables 管理、DynamoDB 表管理(完整版内含)、以及外围的 IAM、CloudWatch、CloudTrail、Billing、Pricing 等运维 server。但 **RDS 本身的控制面不在这里**--见下节。

### 2.3 架构与设计决策

- **认证模式是这组 server 最重要的工程决策**:Aurora 系首选 **RDS Data API**--HTTPS 接口、免直连数据库、天然利用 RDS 的 IAM 集成,凭证只出现在 Data API 的 secret_arn 引用里,不进连接串;同时保留 wire-protocol 直连作为第二路径。Oracle/MSSQL 则是 Secrets Manager 取密 + 数据库原生协议直连。这把"agent 拿数据库凭证"这个最大攻击面压缩到了 IAM 边界内。
- **只读开关**:数据库类 server 普遍提供 `--readonly` 参数。面向 agent 暴露写能力前,先问自己能否接受模型误删。
- **工具描述即接口文档**:每个 server 的 description 写得详细(DynamoDB 甚至分出"设计指导"专用 server),因为模型选工具完全依赖它--这也是 MCP 工具投毒攻击面在本场景的直接映射:从仓库安装的 server 应 pin 版本。
- **本地运行的边界**:stdio 意味着 server 跑在用户机器上,用的是用户的 AWS 凭证,出网流量也过本机--方便开发与调试,但凭证暴露面与审计断层是生产不能接受的直接原因。

### 2.4 控制面主线:aws-rds-mcp 组织解剖(最大的一块拼图)

**定位**:GitHub 组织 [`aws-rds-mcp`](https://github.com/aws-rds-mcp)(2025-06 创建)是 **Amazon RDS 服务团队自己的组织**,与 awslabs/mcp 平级、互不隶属。RDS 控制面 MCP(管理 + 监控诊断)在这里孵化--这就是"AWS 官方数据库 MCP 控制面缺位"谜团的答案:不是没有,而是没搬进主仓。

**源头是 RFC #617**:2025-06-18,团队在 awslabs/mcp 提 RFC《RDS Control Plane MCP Server》,作者 Grace Lee、Jonathan Manuel,mentor Runzhuo Li,manager Mandy Bitar。动机是 RFC #112 里客户抱怨"RDS 管理复杂、监控碎片化"。RFC 明确与 AWS API MCP Server 的三点差异:
1. **破坏性操作走自定义确认 token**(对比 AWS API MCP 的 elicitation 方案,客户端兼容性更广)+ 高风险操作多步校验;
2. **RDS 最佳实践做成默认值**(如 manage-master-password 默认开启)、创建的资源自动打标签便于追踪;
3. **专门化的性能分析**:增强日志分析、指标解读、负载级优化建议。

明确 out of scope:非 RDS 服务、多账号、实时流式指标。

**组织内三个仓库(按时间线)**:

| 仓库 | 创建 | 最后活动 | 定位 | 形态 |
|---|---|---|---|---|
| [`rds-monitoring`](https://github.com/aws-rds-mcp/rds-monitoring) | 2025-06-16 | 2025-08-06 | 只读监控诊断 | 包名 `awslabs.rds-monitoring-mcp-server`,**未上 PyPI** |
| [`rds-management`](https://github.com/aws-rds-mcp/rds-management) | 2025-06-20 | 2025-07-29 | 控制面 CRUD | 包名 `awslabs.rds-management-mcp-server`,**未上 PyPI** |
| [`rds-mcp`](https://github.com/aws-rds-mcp/rds-mcp) | 2025-07-14 | 2025-07-22 | awslabs/mcp 的 fork(+29/-1305),内含**合并版** `src/rds-control-plane-mcp-server` | 目标包名 `awslabs.rds-control-plane-mcp-server` |

**rds-monitoring(只读监控,最后活动 2025-08-06)--以代码为准的完整清单**:

工具 6 个:

| 工具 | 数据源 | 作用 |
|---|---|---|
| `DescribeRDSEvents` | RDS Events | 实例/集群事件(可用区故障、维护、参数变更等) |
| `DescribeRDSPerformanceMetrics` | CloudWatch `get_metric_data` | 性能指标 |
| `DescribeRDSRecommendations` | RDS Recommendations + Trusted Advisor | 优化建议 |
| `FindSlowQueriesAndWaitEvents` | Performance Insights | 慢查询与等待事件分析 |
| `ReadDBLogFiles` | RDS 日志 | 日志读取、过滤、分页 |
| `CreatePerformanceReport` | PI + CloudWatch | 生成性能报告 |

资源模板 9 个(对外只读接口,`aws-rds://` 协议):`db-cluster`、`db-cluster/{cluster_id}`、`db-instance`、`db-instance/{instance_id}`、`db-instance/{dbi}/log`、`db-instance/{dbi_resource_id}/performance_report`(+ `/{report_id}` 读单份)、`{resource_type}/{resource_id}/cloudwatch_metrics`、`metrics_guide`。

运行面:CLI 参数 `--port`(默认 8888)、`--max-items`、`--readonly/--no-readonly`、`--register-resources-as-tools`;默认 streamable HTTP。README 明确"**只读,不含任何管理/修改能力**",给出最小 IAM 清单(rds:Describe*、pi:*、cloudwatch:GetMetricStatistics、support:DescribeTrustedAdvisorChecks)。两个值得抄的机制:**`--register-resources-as-tools`**--资源可整体降级注册为 tool,兼容不支持 resource 的客户端;内置 **rate_limit 装饰器**防 API 刷爆。

**rds-management(控制面 CRUD,最后活动 2025-07-29)--以代码为准的完整清单**:

工具 22 个(注意:README 声称实例参数组可改,代码里实例参数组**只读**):

| 域 | 工具 | 说明 |
|---|---|---|
| 集群(10) | `CreateDBCluster` `ModifyDBCluster` `DeleteDBCluster` `DescribeDBClusters` | CRUD + 查询 |
| | `ChangeDBClusterStatus` `FailoverDBCluster` | 启/停/重启、故障转移 |
| | `CreateDBClusterSnapshot` `DeleteDBClusterSnapshot` `RestoreDBClusterFromSnapshot` `RestoreDBClusterToPointInTime` | 快照与恢复(PITR 在 restore_snapshot.py 内注册) |
| 实例(5) | `CreateDBInstance` `ModifyDBInstance` `DeleteDBInstance` `DescribeDBInstances` `ChangeDBInstanceStatus` | README 写的 ManageDBInstanceStatus 实为 ChangeDBInstanceStatus |
| 参数组(7) | `CreateDBClusterParameterGroup` `ModifyDBClusterParameterGroup` `ResetDBClusterParameterGroup` | 写操作仅集群级 |
| | `DescribeDBClusterParamGroups` `DescribeDBInstanceParamGroups` `DescribeDBClusterParameters` `DescribeDBInstanceParameters` | 集群/实例级均可查 |

资源模板 10 个(README 只写了 4 个,代码更全):`db-cluster`、`db-cluster/{cluster_id}`、`db-cluster/backups`、`db-cluster/{cluster_id}/backups`、`db-instance`、`db-instance/{instance_id}`、`db-instance/backups`、`db-instance/{instance_id}/backups`、`db-cluster/parameter-groups`、`db-cluster/parameter-groups/{name}/parameters`。

运行面:CLI `--port`(默认 8888)、`--max-items`、`--readonly/--no-readonly`、`--region`、`--profile`;默认 streamable HTTP,提供 Dockerfile。安全三板斧用装饰器实现:`readonly_check`(只读模式拦写操作)、`require_confirmation`(高风险操作多步确认,对应 RFC #617 的确认 token 设计)、`handle_exceptions`。**从源码运行**:`uv --directory /path/to/rds-management run main.py`(可带 `--readonly`)。

**rds-mcp fork 里的 rds-control-plane-mcp-server(合并版)**:工具是前两者的并集--集群/实例 CRUD + ReadDBLogFiles + CreatePerformanceReport + DescribeRDSEvents。组织 profile 页([.github](https://github.com/aws-rds-mcp/.github))已经按"RDS Control Plane MCP Server"的统一品牌来写,给出的安装示例是 docker 运行 `awslabs/rds-control-plane-mcp-server:latest`,带 `--readonly`、`--cloudwatch` 等参数,并列出安全特性:IAM 控权、只读模式、破坏性操作影响警告、创建资源自动打标签。署名与 RFC 一致(Grace Lee、Jonathan Manuel,mentor Runzhuo Li,manager Mandy Bitar)。

**现状与判断(2026-10-19 核实)**:
- **未并入 awslabs/mcp**:`src/rds-control-plane-mcp-server` 在主仓 404,根 README 0 提及,server-cards.json 无条目;
- **未发 PyPI**:`awslabs.rds-control-plane-mcp-server` / `awslabs.rds-management-mcp-server` / `awslabs.rds-monitoring-mcp-server` 均 404;DockerHub 公开镜像亦未能验证到;
- **组织整体停更**:三个仓库最后活动停在 2025-07-22 ~ 2025-08-06;
- 结论:它要么是在 Agent Toolkit 转型期被搁置,要么未来以 Agent Toolkit 形态重新发布。**今天是"未 GA 的团队仓库"状态**--要用只能 clone 源码跑,生产接入前 pin commit、自审代码、配最小 IAM。它的设计文档(RFC #617)和代码仍是理解"控制面 MCP 该怎么做"的最佳样本。

### 2.5 数据面解剖:postgres-mcp-server 的工具箱

以 Aurora PostgreSQL server 为样本(`src/postgres-mcp-server`,2026-10-19 的 main),数据面 server 的工具面比想象中小--**7 个工具,两类职责**:

| 工具 | 作用 |
|---|---|
| `run_query` | 执行 SQL 返回行结果,支持参数化查询 |
| `get_table_schema` | 取表结构:列名、数据类型、列注释 |
| `connect_to_database` | 建立连接并缓存在 server 内部 |
| `is_database_connected` | 检查连接是否已建立 |
| `get_database_connection_info` | 列出所有已缓存连接 |
| `create_cluster` | 创建 Aurora PostgreSQL 集群(serverless v2 或 express 配置) |
| `get_job_status` | 轮询后台任务状态(建集群是异步长任务) |

几个值得抄的设计:

- **连接方式三选一**:`rdsapi`(RDS Data API,仅 APG)、`pgwire`(直连,APG/RPG 均可)、`pgwire_iam`(IAM 认证直连,仅 APG)。server 同时缓存多 endpoint 连接,靠 LLM 在提示里声明式切换,而不是每查一次传一串连接参数。
- **`run_query` 的 SQL 策略护栏**:基于 pglast 解析器做白盒判定--只读模式拒绝写语句;解析失败、超大输入、多语句提交一律 fail-closed。README 自己承认 `--allow_write_query` 的只读 enforcement 是 best-effort,所以有函数级黑白名单兜底:`nextval()`/`setval()`/`pg_stat_reset*()`/`cron.schedule()` 拒绝,`currval()`/`pg_prewarm()` 放行。
- **`create_cluster` 一条龙**:建 serverless v2 集群后自动给当前用户配 IAM 策略、建好连接缓存;express 模式强制 IAM。因为建集群耗时,实际跑后台 worker,用 `get_job_status` 轮询。这把"数据面 server"的边界扩到了控制面,是 awslabs/mcp 里少数自带建资源能力的数据库 server。
- **直连强制 TLS**,默认 `verify-full`;另有最小权限护栏 `--privilege_check`(检测到 superuser/rds_superuser/BYPASSRLS 角色时 warn 或 enforce 拒绝)。

一句话:数据面靠 `run_query` + `get_table_schema` 两个工具打天下,其余五个管连接生命周期和建集群。

**postgres-mcp-server 的仓内配套物:Kiro Power**。仓库自带 `kiro_power/` 与 `kiro_proj_steering/` 两组文件,是 Amazon 自家 Kiro 客户端的专用配套,不是通用 Skill:`kiro_power/POWER.md`(10.8KB)声明一个名为 `amazon-aurora-postgresql` 的 Kiro Power--frontmatter 明写它把 Aurora PostgreSQL MCP server 的**数据面操作(查询/建表/schema 管理)和控制面操作(建集群)打包绑定**,并挂两份 steering 文件(`aurora-postgres-mcp`:MCP server 用法/SQL 风格指南;`aurora-postgres`:schema 设计/索引/查询优化/迁移的完整开发指南);`kiro_proj_steering/tech.md` 是项目级技术栈约定(如建集群 5-7 分钟、用 `get_job_status` 每分钟轮询、连接首选 RDS Data API)。这是"一个数据库产品 = MCP server + steering 知识 + 客户端绑定"的标准封装样本,但目前只服务 Kiro 一家。

## 2.6 配套 Skill 层：Agent Toolkit for AWS（MCP 的"脑"）

先回答本次调研的原始问题：**`aws-rds-mcp/rds-management` 和 `awslabs/mcp` 的 postgres-mcp-server，各自的配套 Skill 是什么、在哪**。结论分两层：

1. **两个仓库本体都不带 Skill**。rds-management 全仓只有 README/CHANGELOG/tests（2026-10-20 用 GitHub API 递归列树核实），grep 无 skill 字样；postgres-mcp-server 无 SKILL.md，仓内配套物只有上节说的 Kiro 专用文件。
2. **官方配套 Skill 住在第三个仓库**：[`aws/agent-toolkit-for-aws`](https://github.com/aws/agent-toolkit-for-aws)（README 带 GA 状态徽章），路径 `skills/`。这才是 "Agent Toolkit 是 MCP servers、plugins、skills 的继任者" 这句话里 skills 的实体。

### 体系结构：一个总路由 + 15 个专科

Agent Toolkit 的定位一句话："Help AI coding agents build, deploy, and manage applications on AWS"，兼容 Claude Code / Codex / Cursor / Kiro / fx 五家客户端。Skill 层分两类：

- **Core skills**（随 aws-core 插件分发）——与数据库直接相关的是 `aws-database`：**总路由 Skill，当前 v3**。它的工作方式值得逐字引用："**STOP — Do not answer from general knowledge.**" 任何数据库问题先到子技能注册表做意图匹配，命中就**必须**加载对应专科 Skill 再回答；事实类问题（价格/配额/GA 状态）禁止凭训练数据作答，按优先级查 knowledge card → AWS MCP 文档工具 → 服务 `llms.txt`。它还内置反方机制：用户 pushback 匹配 `report-issue` 触发词（"that's wrong" 等）时必须改道复核，"目标是正确答案，不是与你第一次的回答保持一致"。这是把"模型幻觉治理"写成 Skill 工程规范的样本。
- **Specialized skills**：`skills/specialized-skills/database-skills/` 下 **15 个数据库专科**，2026-10-20 核实的完整清单：

| Skill | 覆盖对象 | 与本文两条 MCP 线的关系 |
|---|---|---|
| `rds-oss` | RDS MySQL/MariaDB/PostgreSQL | **rds-management MCP 的正配**：决策与护栏 |
| `rds-oracle` / `rds-sqlserver` / `rds-db2` | RDS 商业引擎 | 同上，按引擎分科 |
| `amazon-aurora-mysql` / `amazon-aurora-postgresql` | Aurora 两引擎 | **postgres-mcp-server 的正配**：管库决策 |
| `amazon-documentdb` / `amazon-dynamodb` / `amazon-keyspaces` / `amazon-neptune` / `amazon-elasticache` | 文档/键值/宽列/图/缓存 | awslabs/mcp 各数据面 server 的决策层 |
| `aurora-dsql` / `timestream-influxdb` | 分布式 SQL / 时序 | 同上 |
| `creating-amazon-aurora-db-cluster-with-instances` / `exporting-rds-to-s3` | 两个场景型工作流 | 横跨 MCP 与 Skill 的任务包 |

### 与 rds-management 的配套关系：脑/手分离，证据在原文里

`rds-oss` 是这个体系的样板，拆开来能看到"MCP 的脑"长什么样：

- **五个决策域**：实例创建、升级规划、承诺定价（RI vs DSP）、RDS Proxy 评估、蓝绿部署。开头声明 "Produces cost estimates, precheck findings, and CLI commands"，且变更类操作 "advisory only, **never executes without explicit user confirmation**"。
- **路由表**：Decision Guide 把用户问题逐类映射到 references/ 下的工作流文件（升级走 `upgrade-workflow.md`、定价走 `commitment-pricing-workflow.md`……），README 写明"宽泛请求就把五个选项各列一行让用户选"。
- **防混淆护栏**：RDS vs Aurora 八行对照表（LTS、Serverless、I/O-Optimized、Data API、实例 vs 集群拓扑、DSP 期限……），明令 "You MUST NOT apply Aurora concepts to RDS"，发现实例其实是 Aurora 就停手改道。
- **生产创建的 11 条强制最佳实践**：最新稳定大版本、Multi-AZ、CMK 加密、禁公网、7 天备份保留、Performance Insights 7 天、删除保护、**禁用 admin/root/postgres 等默认主用户名**、Secrets Manager 托管密码（"Do NOT accept or use a plaintext password"）、gp3、以及**自动打标签** `created_by=rds-oss-skill` + `generation_model`——最后这条正是 RFC #617 给 rds-management 设计的能力，出现在 Skill 层。
- **与 MCP 的关系，原文一句定音**："The AWS MCP server is recommended for executing commands **but is not required**; all operations can also be performed via the AWS CLI." Skill 自带手和脚，MCP 只是更顺的那只手。

### 与 postgres-mcp-server 的配套关系：数据面查数，Skill 面管库

`amazon-aurora-postgresql` Skill 采用**子技能注册表**架构（SKILL.md 里挂一张路由表，意图命中后按需 `file_read` 加载对应 `references/{id}-instructions.md`，只加载需要的上下文）：

- 注册表条目（2026-10-20 核实）：`create`（建集群）、`express-create`（Express 配置快速路径）、`serverless-advisory`（ACU 容量/scale-to-zero）、`io-optimized`（存储类型评估，25% I/O 成本阈值规则）、`commitment-pricing`、`upgrade-planning`。
- **Express-first 路由**：Aurora PG 建集群默认走 express（单 API 调用、无客户 VPC、**经 Internet Access Gateway 的 IAM-only 认证、无 master password**），只有触发客户 VPC/CMK/自定义参数组任一条件才升级全配置路径——且 "Don't present the choice to the user — decide, then state which path and why"。
- **安全护栏分级 + 执行偏好**：变更请求先按护栏等级分类、与用户确认、打资源标签，然后 "execute (**MCP-preferred, CLI fallback**)"。

组合起来的分工：**postgres-mcp-server 回答"库里的数据是什么"（run_query），amazon-aurora-postgresql Skill 回答"这个库该怎么建/改/省/升"（六张子技能路由）**；前者是数据面，后者是控制面决策。一个 Aurora PG 对象，左右手各一只。

### 分发渠道与松耦合证据

分发有四条路（skills/README.md 原文）：**AWS CLI**（`aws configure agent-toolkit`）；**Claude Code 官方插件市场**（`aws-core` / `aws-agents` / `aws-data-analytics` / `aws-agents-for-devsecops`，挂在 claude-plugins-official）；**本地安装**（`npx skills add aws/agent-toolkit-for-aws/skills`，各家 agent 的 skills 目录路径都列了表）；**运行时按需发现**——让 agent 通过 AWS MCP Server 搜 "Search for AWS skills related to databases"，Skill 以 MCP resource 形态被检索加载，无需本地安装。

松耦合是双向核实过的：rds-management 的 README 全文不提 skill / agent-toolkit，agent-toolkit 的 README 全文不提 aws-rds-mcp。两边代码互不引用，但语义上严丝合缝——**RFC #617 的两块招牌就这样分家了：确认 token 与只读护栏留在 rds-management 的代码里（手），最佳实践默认值与自动打标签进了 agent-toolkit 的 Skill 里（脑）**。组织 2025-08 停更而 Skill 层持续迭代到 v3，佐证了判断：RDS 控制面 MCP 的遗产更可能以 Agent Toolkit 形态延续，aws-rds-mcp 仓库本身转正的概率在下降。

---

## 三、托管路线：Bedrock AgentCore 栈

AgentCore 是 AWS 的 agent 平台(2025 re:Invent 发布,现已含 12 个模块服务),与"数据库 MCP"直接相关的有五个:

- **Gateway(核心)**:全托管 AI 网关,一个入口承接 agent 流量。三种工具来源:OpenAPI 规范、Smithy 模型、Lambda 函数--几行代码转成 MCP 兼容工具;同时支持透传已有 MCP server 和 A2A 流量、跨模型 provider 的推理路由。关键能力:**入站认证 + 出站认证双向管理**(OAuth 流程、token 刷新、凭证保管,"唯一同时覆盖两者"的全托管服务)、语义工具选择(上千工具时压缩 prompt)、一键集成 Salesforce/Slack/Jira 等。对数据库场景的含义:你的内网数据库 API 不需要写 MCP server,用 OpenAPI 描述交给 Gateway 即可变成 MCP 工具。
- **Registry**:组织内 agent、MCP server、工具、skill 的目录,带发布/审核/批准工作流和语义搜索--解决"团队里 50 个 MCP server 谁先批谁后用"。
- **Policy**:用自然语言或 Dogwood 策略语言(兼容 Cedar)写规则,在 Gateway 上**拦截每一次工具调用**--哪个 agent 能调哪个工具、什么条件下可执行。这是托管路线对"数据面安全"的正式回答。
- **Identity**:agent 身份与认证(Cognito/Okta/Entra ID/Auth0),替代手工分发凭证。
- **Payments**:通过 x402 / MPP 协议给付费 MCP server 和内容做微支付--数据库 API 商业化计费的管道,值得留意。

另有 **AWS MCP Server(2026-05-06 GA)**:AWS 官方托管的远程 MCP server,2025-11 preview、2026-05-06 正式 GA--GA 版支持 **IAM context keys**(不再需要为它单独配置 IAM 权限,可用条件键做细粒度管控),预置 Agent SOPs(按 AWS 最佳实践的多步任务流程)、语法校验的 API 调用、零凭证暴露、CloudTrail 全审计。这是"托管路线"在官方服务本身的样板。**AWS Knowledge MCP Server** 是第二个托管 remote(文档与知识)。注意区分:AWS MCP Server 走"广覆盖 API + 托管",aws-rds-mcp 走"RDS 专科 + 源码分发",二者设计取向不同。

## 四、三条线对比

| 维度 | awslabs/mcp(开源) | aws-rds-mcp(开源,团队仓) | AgentCore 托管 |
|---|---|---|---|
| 运行位置 | 用户本机 / 自行部署 | 用户本机(源码运行) | AWS 全托管 serverless |
| 传输 | 仅 stdio(迁 Streamable HTTP 中) | 仅 stdio(uv run / docker) | Streamable HTTP 端点 |
| 凭证 | 本机 AWS 凭证 / secret_arn | 本机 AWS 凭证 | 零凭证暴露,IAM + Identity |
| 审计 | 无(本机进程) | 无(本机进程) | CloudTrail / CloudWatch 全链路 |
| 数据面保护 | `--readonly` 开关(自律) | 默认只读 + 破坏操作确认 token(自律) | Policy 规则(他律,逐调用拦截) |
| 覆盖面 | 20+ 数据库数据面一等公民 | 仅 RDS 控制面 + 监控诊断 | 任意 OpenAPI/Lambda 后端,数据库需自行包装 |
| 配套 Skill | **无(本体不带);正配在 Agent Toolkit(GA,持续迭代)** | **同左;rds-oss 是其正配** | Registry 同时管 server 与 skill 目录;AWS MCP Server 可运行时检索 Skill |
| 分发 | uvx / GitHub / Docker | **仅 GitHub 源码** | Registry 治理 + 语义发现 |
| 成熟度 | 活跃维护,9.8k stars | 2025-08 停更,未 GA,未入 PyPI | AWS MCP Server 已 GA;Gateway 持续扩模块 |
| 成本 | 免费(开源) | 免费(开源) | 按量计费 |

判断:三条线不是替代关系而是分工--awslabs/mcp 是"数据面参考实现 + 开发体验入口",aws-rds-mcp 是"控制面专科实现 + 尚未转正",Gateway 是"生产化的共同答案"。真正的张力在**数据面**:开源侧把写能力以 `--readonly` 开关交给你自己把关,托管侧用 Policy 把把关产品化。对合规团队,这不是选择题。

## 五、风险与注意事项

1. **MCP 通用风险在本场景放大**:协议层面的已知攻击(工具描述投毒、rug pull--恶意 server 在工具列表落地后替换实现)对数据库 server 尤其危险--一个被污染的 Redshift server 等于数据仓库凭据。pin 版本、审查 description、生产只接内部包装的 server。
2. **开源路线的凭证断层**:stdio server 用你本机的凭证执行,公司审计视角里这些操作"不是人就是 agent",Agent Toolkit 的 IAM 条件键正是补这个洞--过渡期建议显式配置。
3. **写能力的默认开放问题**:多数数据面 server 未默认只读,agent 拿到 write 工具后的破坏半径取决于你的 IAM 粒度。最小权限原则:给 server 专用 IAM 角色,只授目标库必要动作。rds-management 这类控制面 server 的破坏半径是整库删实例,比数据面更危险,`--readonly` 应视为必选项而非可选项。
4. **团队仓的供应链风险**:aws-rds-mcp 三个仓库未入 PyPI、无签名发布物、2025-08 起停更--引用它等于把一段未经 AWS 发行流程审视的代码接进你的凭证边界。用可以,pin commit + 代码评审 + 独立 IAM 角色,三件套缺一不可。
5. **托管路线的锁定**:Gateway 的 OpenAPI 转换是好用,但工具目录、Policy、Identity 都在 AgentCore 内沉淀,跨云迁移成本随使用深度上升。
6. **文档口径漂移**:awslabs/mcp 根 README 与其 src README 之间存在已知不一致(MSSQL 认证方式、OpenSearch 去向)--引用任何一条前先落到 src README 核对;本文已按 src README 修正。
7. **Skill 层的信任边界**:Skill 本质是注入到模型上下文里的指令集,`description` 字段是模型的触发依据,也是新的投毒攻击面--从第三方渠道安装 Skill 与安装 MCP server 同等对待(pin 版本、审 frontmatter);另外 Skill 声称"超越训练数据",但知识卡本身有版本(v3),过期 Skill 给出过时限额/价格的概率不低,生产使用前核对 knowledge card 日期。

## 六、动手路径

1. **本机十分钟**:装好 `uv`,在 Claude Code / Cursor 配置里加一条 Aurora PostgreSQL server(`uvx awslabs.postgresql-mcp-server@latest --resource_arn ... --secret_arn ... --readonly true`),用自然语言查一次表结构。
2. **RDS 控制面体验**:clone `aws-rds-mcp/rds-management`,`uv --directory rds-management run main.py --readonly` 接进 MCP 客户端,让 agent 列一次集群/实例、读一份慢查询报告;想试合并版就 clone `aws-rds-mcp/rds-mcp` 跑 `src/rds-control-plane-mcp-server`。生产前置条件:pin commit、自审、专用 IAM 角色。
3. **团队共享**:把内部数据库维护团队的 Python 脚本用 FastMCP 包成只读 server 提交内审,或用 Gateway 把现有 OpenAPI 服务转 MCP--后者不用写 MCP 代码。控制面场景优先评估 AWS API MCP Server(广覆盖、elicitation 确认)与 rds-control-plane(专科、token 确认)哪个更适配你的客户端。
4. **生产化**:server 进 AgentCore Registry 走审批流;Gateway 前挂 Policy(先全只读试运行);Identity 接 IdP;观测接 CloudWatch。写能力单独审批、单独角色、单独 Policy。
5. **迁移观察项**:盯 Agent Toolkit for AWS 的功能矩阵(aws-rds-mcp 的能力正以其 Skill 形态重现,rds-oss 的自动打标签即 RFC #617 遗产)与 awslabs/mcp 的归档计划,新投入以 Agent Toolkit 为准。
6. **体验配套 Skill**(十分钟):任选一路--Claude Code 里 `/plugin install aws-core@claude-plugins-official`,或 `npx skills add aws/agent-toolkit-for-aws/skills`,或配好 AWS MCP Server 后直接问 "Search for AWS skills related to databases"。然后问一句"我该用 RDS PostgreSQL 还是 Aurora"--看 `aws-database` 路由到哪个专科、`rds-oss` 的防混淆表怎么工作。再进一步:让 agent 走一遍 rds-oss 的升级规划工作流,观察它是否遵守"advisory only, never executes without explicit user confirmation"。

## 附录

**A. 开源数据面 server 与访问方式速查**:RDS Data API(Aurora PG/MySQL,或 wire 直连)、Secrets Manager 取密直连(Oracle、MSSQL)、直连协议(DSQL、DocumentDB、Neptune、Keyspaces、Timestream、Valkey、Memcached)、AWS API(DynamoDB、S3 Tables、ElastiCache 控制面、SiteWise、Glue/EMR/Athena)、Lambda 包装(Lambda Tool)。OpenSearch 已移交社区项目 `opensearch-project/opensearch-mcp-server-py`。

**B. aws-rds-mcp 组织时间线**:2025-06-16 组织内首个仓库 rds-monitoring 创建;2025-06-18 awslabs/mcp RFC #617 发布;2025-06-20 rds-management 创建;2025-07-14 rds-mcp fork 创建(内含合并版 rds-control-plane-mcp-server);2025-07-22 fork 最后 push;2025-07-29 rds-management 最后提交;2025-08-06 rds-monitoring 最后更新;截至 2026-10-19 未并入 awslabs/mcp、未上 PyPI。

**C. 关键日期**:2025-03-21 awslabs/mcp 创建;2025-05-26 SSE 支持移除;2025-11 AWS MCP Server preview;2025 re:Invent AgentCore 发布;2026-05 Agent Toolkit for AWS 上线;2026-05-06 AWS MCP Server GA(IAM context keys);2026-10-19 awslabs/mcp 约 9.8k stars。

**D. 与 MCP 协议演进的咬合**:awslabs/mcp 从 SSE 撤向 Streamable HTTP,与协议 2026-07-28 版的无状态核心同向;托管 Gateway 天然适配无状态 per-request 模型。

## 参考资料

- [GitHub: awslabs/mcp(开源 server 全清单、stdio 传输说明、SSE 移除公告、Agent Toolkit 继任者提示,2026-10-19 抓取)](https://github.com/awslabs/mcp)
- [GitHub: awslabs/mcp src/postgres-mcp-server(数据面工具箱解剖:7 工具、SQL 护栏、连接三模式,2026-10-19 核对)](https://github.com/awslabs/mcp/tree/main/src/postgres-mcp-server)
- [GitHub: awslabs/mcp src/mssql-mcp-server(认证方式以 src README 为准,2026-10-19 核对)](https://github.com/awslabs/mcp/tree/main/src/mssql-mcp-server)
- [GitHub: aws-rds-mcp/rds-management(RDS 控制面 CRUD,22 工具,2026-10-19 抓取)](https://github.com/aws-rds-mcp/rds-management)
- [GitHub: aws-rds-mcp/rds-monitoring(RDS 只读监控,6 工具,2026-10-19 抓取)](https://github.com/aws-rds-mcp/rds-monitoring)
- [GitHub: aws-rds-mcp/rds-mcp(awslabs/mcp fork,内含合并版 rds-control-plane-mcp-server,2026-10-19 核对)](https://github.com/aws-rds-mcp/rds-mcp)
- [RFC #617: RDS Control Plane MCP Server(设计文档:确认 token、最佳实践默认值、out of scope,2026-10-19 抓取)](https://github.com/awslabs/mcp/issues/617)
- [AWS What's New: AWS MCP Server 正式 GA(2026-05-06,IAM context keys)](https://aws.amazon.com/about-aws/whats-new/2026/05/aws-mcp-server/)
- [AWS 文档:What is Amazon Bedrock AgentCore(12 个模块服务全景:Gateway/Registry/Policy/Identity/Payments 等,2026-10 抓取)](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/what-is-bedrock-agentcore.html)
- [AWS 文档:AgentCore Gateway(OpenAPI/Smithy/Lambda 转 MCP、双向认证、语义工具选择、目标类型,2026-10 抓取)](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway.html)
- [GitHub API: awslabs/mcp 仓库元数据(9.8k stars、2025-03-21 创建、Apache-2.0、最近 push 2026-10-16,2026-10-19 查询)](https://api.github.com/repos/awslabs/mcp)
- [GitHub: aws/agent-toolkit-for-aws(Skill 层主体仓库,GA;skills/README.md 列出四种分发渠道,2026-10-20 抓取)](https://github.com/aws/agent-toolkit-for-aws)
- [GitHub: agent-toolkit-for-aws skills/core-skills/aws-database(总路由 Skill v3:"STOP — do not answer from general knowledge"、子技能注册表、防幻觉规范,2026-10-20 抓取)](https://github.com/aws/agent-toolkit-for-aws/tree/main/skills/core-skills/aws-database)
- [GitHub: agent-toolkit-for-aws skills/specialized-skills/database-skills/rds-oss(rds-management 正配:五决策域、11 条生产创建强制项、RDS vs Aurora 防混淆表,2026-10-20 抓取)](https://github.com/aws/agent-toolkit-for-aws/tree/main/skills/specialized-skills/database-skills/rds-oss)
- [GitHub: agent-toolkit-for-aws skills/specialized-skills/database-skills/amazon-aurora-postgresql(postgres-mcp-server 正配:子技能注册表、express-first 路由、MCP-preferred 执行,2026-10-20 抓取)](https://github.com/aws/agent-toolkit-for-aws/tree/main/skills/specialized-skills/database-skills/amazon-aurora-postgresql)
- [GitHub: awslabs/mcp src/postgres-mcp-server/kiro_power/POWER.md(仓内 Kiro Power 配套:把数据面/控制面 MCP 操作与 steering 指南打包绑定,2026-10-20 抓取)](https://github.com/awslabs/mcp/tree/main/src/postgres-mcp-server/kiro_power)
