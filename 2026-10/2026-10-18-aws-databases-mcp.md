# AWS 数据库 MCP 全景：开源工具箱与托管网关的两条路线

> **TL;DR**：AWS 的数据库 MCP 版图分两层——开源侧是 `awslabs/mcp` 仓库（9.8k stars，2025-03 创建，Apache-2.0）里 20+ 个仅支持 stdio 的本地 server，覆盖 DynamoDB、Aurora（PG/MySQL/DSQL）、Redshift、DocumentDB、Neptune、Keyspaces、Timestream、Oracle/MSSQL、Valkey/Memcached、S3 Tables 及 Glue/EMR/Athena 数据管道，分发方式是 uvx + FastMCP，认证走 RDS Data API 或直连 + Secrets Manager；**控制面（RDS 管理）另有主线：RDS 服务团队在独立组织 `aws-rds-mcp` 孵化了 rds-monitoring / rds-management / rds-control-plane 三个 MCP server（RFC #617），但截至 2026-10 尚未并入 awslabs/mcp，也未上 PyPI，只能从源码运行**；托管侧是 Bedrock AgentCore 的 Gateway（把 OpenAPI/Smithy/Lambda 转成 MCP 工具的双向认证网关）+ Registry + Policy，以及 **2026-05 已 GA** 的官方托管 AWS MCP Server（IAM context keys、零凭证暴露、CloudTrail 全审计）。选型一句话：本地开发接开源 server，RDS 控制面用 aws-rds-mcp（接受"未 GA 团队仓库"的前提），生产共享工具走 Gateway 托管，数据面操作务必开 `--readonly` 或套 Policy。
> **数据截至 2026-10-19。** 开源侧以 `awslabs/mcp` 仓库 README 与各 server src README 为准，aws-rds-mcp 侧以组织内三个仓库与 awslabs/mcp RFC #617 为准，托管侧以 AWS 官方公告与文档为准。

---

你的二分法是对的，而且比表面更深一层：AWS 在数据库 MCP 这件事上同时跑着**两条产品路线**——一条把 20 多个数据库的 MCP server 当开源软件发（本地 stdio、自己跑、自己管凭证），一条把 MCP 当托管云服务卖（AgentCore Gateway 托管端点、按量计费、IAM 集成）。前者解决"我的开发机上 agent 能不能用数据库"，后者解决"全公司的 agent 怎么安全地共享同一批工具"。两条路线的技术取舍几乎处处相反。

但还有第三块拼图不在任何一条路线里：**RDS 服务团队自己的 GitHub 组织 `aws-rds-mcp`**——控制面 MCP（建删改实例/集群/参数组/快照 + 监控诊断）的真正主角在这里孵化，不在 awslabs/mcp 主仓。搞清楚这个组织的来龙去脉，才能解释为什么"AWS 官方数据库 MCP"看起来控制面缺位。

本文的路线：先盘点开源 server 清单与架构，再专章解剖 aws-rds-mcp 组织，然后拆托管 AgentCore 栈，接着对比路线，最后给选型和动手路径。

一句话心智模型：**开源路线是"把数据库的钥匙交给本机的 agent"，托管路线是"把数据库放在 agent 够得着但有门卫的柜台后面"，而 aws-rds-mcp 是"RDS 团队自己打造的仪表盘，还没搬进 4S 店"。**

---

## 一、定义与边界

- **调研对象**：AWS 官方（awslabs / AWS 服务团队）发布的数据库相关 MCP server 与托管 MCP 能力；不含社区第三方 server，不含非数据库类（仅在与数据库场景交叉时提及）。
- **"管理服务 vs 数据服务"的口径**：管理服务 = 控制面操作（建删实例/集群、改配置、参数组、快照、扩节点、查监控）；数据服务 = 数据面操作（查询、读写、执行 SQL）。这个二分贯穿全文，因为两条路线在两者上的取舍不同。
- **信息源**：一手（GitHub 仓库、AWS 官方文档、AWS 公告）为主；所有数字带日期。注意 awslabs/mcp 根 README 的表格偶有与其 src 下各 server README 不一致之处，本文以 src README 为准并逐条标注。

## 二、开源路线：awslabs/mcp 仓库解剖

### 2.1 基本面

- 仓库 `awslabs/mcp`，2025-03-21 创建，Apache-2.0，截至 2026-10-19 约 9.8k stars / 1.8k forks，最近 push 2026-10-16，仍在活跃维护。
- **传输：仅 stdio**。2025-05-26 起全部移除 SSE 支持，正在向 Streamable HTTP 迁移——与 MCP 协议本身的演进节奏一致（协议 2026-07 版核心已转向无状态、每请求自携带协商信息）。
- **技术栈**：Python + FastMCP 框架，通过 `uvx` 一键运行，例如 `uvx awslabs.mysql-mcp-server@latest`。Cursor、VS Code、Kiro、Claude Code、Windsurf、Amazon Q CLI 都有对应的一键安装配置；也支持 Docker 运行。
- **官方已发出迁移信号**：README 顶部公告称 "Agent Toolkit for AWS 已上线，是 MCP servers、plugins、skills 的继任者"（2026-05 发布）——增加了 IAM 条件键（区分 agent 操作与人类操作）、CloudWatch/CloudTrail 可见性、经过准确性评估的 skills。awslabs/mcp 继续维护但新重心在 Agent Toolkit。
- 仓库还在持续长出新 server（Amazon Translate、AppSync、Network、SAP 管理、SageMaker Spark 排障/升级等），数据库主清单以 README "SQL & NoSQL Databases" 一节为准。

### 2.2 数据库 server 清单（按"管理服务/数据服务"重排）

**数据服务（数据面）**：

| Server | 数据访问方式 | 备注 |
|---|---|---|
| Aurora PostgreSQL | RDS Data API 或 pgwire 直连 | 支持多 endpoint，LLM 提示切换连接方式 |
| Aurora MySQL | RDS Data API 或 mysqlwire 直连 | 也支持 RDS MySQL 与自建 MySQL/MariaDB |
| Aurora DSQL | 直连 | 分布式 SQL，PG 兼容 |
| RDS Oracle | Secrets Manager 取密 + python-oracledb thin 直连 | 免 Oracle Instant Client |
| RDS MSSQL | Secrets Manager 取密直连 | **根 README 表格写"via RDS Data API"是错的，以其 src README 为准** |
| DynamoDB | AWS API | 两个版本：设计指导/数据建模版 + 完整表管理版 |
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

**两个边界变化（2026-10 核实）**：
- **OpenSearch 已移出 awslabs/mcp**：`src/opensearch-mcp-server` 目录不存在（404），根 README 仅保留一条外链，指向社区项目 `opensearch-project/opensearch-mcp-server-py`。它不再是 AWS 官方维护的 server。
- **MSK 没有 MCP server**：当前 README 与 src 均无 MSK 条目（只有 Amazon MQ）。早前部分资料里的 MSK 说法不要沿用。

**管理服务（控制面）**：ElastiCache（完整控制面操作）、S3 Tables 管理、DynamoDB 表管理（完整版内含）、以及外围的 IAM、CloudWatch、CloudTrail、Billing、Pricing 等运维 server。但 **RDS 本身的控制面不在这里**——见下节。

### 2.3 架构与设计决策

- **认证模式是这组 server 最重要的工程决策**：Aurora 系首选 **RDS Data API**——HTTPS 接口、免直连数据库、天然利用 RDS 的 IAM 集成，凭证只出现在 Data API 的 secret_arn 引用里，不进连接串；同时保留 wire-protocol 直连作为第二路径。Oracle/MSSQL 则是 Secrets Manager 取密 + 数据库原生协议直连。这把"agent 拿数据库凭证"这个最大攻击面压缩到了 IAM 边界内。
- **只读开关**：数据库类 server 普遍提供 `--readonly` 参数。面向 agent 暴露写能力前，先问自己能否接受模型误删。
- **工具描述即接口文档**：每个 server 的 description 写得详细（DynamoDB 甚至分出"设计指导"专用 server），因为模型选工具完全依赖它——这也是 MCP 工具投毒攻击面在本场景的直接映射：从仓库安装的 server 应 pin 版本。
- **本地运行的边界**：stdio 意味着 server 跑在用户机器上，用的是用户的 AWS 凭证，出网流量也过本机——方便开发与调试，但凭证暴露面与审计断层是生产不能接受的直接原因。

### 2.4 控制面主线：aws-rds-mcp 组织解剖（最大的一块拼图）

**定位**：GitHub 组织 [`aws-rds-mcp`](https://github.com/aws-rds-mcp)（2025-06 创建）是 **Amazon RDS 服务团队自己的组织**，与 awslabs/mcp 平级、互不隶属。RDS 控制面 MCP（管理 + 监控诊断）在这里孵化——这就是"AWS 官方数据库 MCP 控制面缺位"谜团的答案：不是没有，而是没搬进主仓。

**源头是 RFC #617**：2025-06-18，团队在 awslabs/mcp 提 RFC《RDS Control Plane MCP Server》，作者 Grace Lee、Jonathan Manuel，mentor Runzhuo Li，manager Mandy Bitar。动机是 RFC #112 里客户抱怨"RDS 管理复杂、监控碎片化"。RFC 明确与 AWS API MCP Server 的三点差异：
1. **破坏性操作走自定义确认 token**（对比 AWS API MCP 的 elicitation 方案，客户端兼容性更广）+ 高风险操作多步校验；
2. **RDS 最佳实践做成默认值**（如 manage-master-password 默认开启）、创建的资源自动打标签便于追踪；
3. **专门化的性能分析**：增强日志分析、指标解读、负载级优化建议。

明确 out of scope：非 RDS 服务、多账号、实时流式指标。

**组织内三个仓库（按时间线）**：

| 仓库 | 创建 | 最后活动 | 定位 | 形态 |
|---|---|---|---|---|
| [`rds-monitoring`](https://github.com/aws-rds-mcp/rds-monitoring) | 2025-06-16 | 2025-08-06 | 只读监控诊断 | 包名 `awslabs.rds-monitoring-mcp-server`，**未上 PyPI** |
| [`rds-management`](https://github.com/aws-rds-mcp/rds-management) | 2025-06-20 | 2025-07-29 | 控制面 CRUD | 包名 `awslabs.rds-management-mcp-server`，**未上 PyPI** |
| [`rds-mcp`](https://github.com/aws-rds-mcp/rds-mcp) | 2025-07-14 | 2025-07-22 | awslabs/mcp 的 fork（+29/-1305），内含**合并版** `src/rds-control-plane-mcp-server` | 目标包名 `awslabs.rds-control-plane-mcp-server` |

**rds-monitoring（只读监控）**：6 个工具——DescribeRDSEvents（事件）、DescribeRDSPerformanceMetrics（CloudWatch 指标）、DescribeRDSRecommendations（含 Trusted Advisor 检查）、FindSlowQueriesAndWaitEvents（Performance Insights 慢查询/等待事件）、ReadDBLogFiles（日志读取过滤分页）、CreatePerformanceReport（性能报告）。资源模板覆盖 cluster/instance 列表与详情、日志、性能报告、cloudwatch_metrics、metrics-guide。README 明确"**只读，不含任何管理/修改能力**"，并给出最小 IAM 清单（rds:Describe*、pi:*、cloudwatch:GetMetricStatistics、support:DescribeTrustedAdvisorChecks）。

**rds-management（控制面 CRUD）**：22 个工具——集群 10 个（Create/Modify/Delete/ChangeDBClusterStatus 启停重启/Failover/CreateDBClusterSnapshot/DeleteDBClusterSnapshot/RestoreDBClusterFromSnapshot/RestoreDBClusterToPointInTime/DescribeDBClusters）、实例 5 个（Create/Modify/Delete/ManageDBInstanceStatus/DescribeDBInstances）、参数组 8 个（集群/实例各 4 个：Create/Modify/Reset/Describe）。资源模板：db-cluster、db-cluster/{id}、db-instance、db-instance/{id}。CLI 参数：`--readonly/--no-readonly`、`--max-items`、`--port`、`--region`、`--profile`。**从源码运行**：`uv --directory /path/to/rds-management run main.py`（可带 `--readonly`），也提供 Dockerfile。

**rds-mcp fork 里的 rds-control-plane-mcp-server（合并版）**：工具是前两者的并集——集群/实例 CRUD + ReadDBLogFiles + CreatePerformanceReport + DescribeRDSEvents。组织 profile 页（[.github](https://github.com/aws-rds-mcp/.github)）已经按"RDS Control Plane MCP Server"的统一品牌来写，给出的安装示例是 docker 运行 `awslabs/rds-control-plane-mcp-server:latest`，带 `--readonly`、`--cloudwatch` 等参数，并列出安全特性：IAM 控权、只读模式、破坏性操作影响警告、创建资源自动打标签。署名与 RFC 一致（Grace Lee、Jonathan Manuel，mentor Runzhuo Li，manager Mandy Bitar）。

**现状与判断（2026-10-19 核实）**：
- **未并入 awslabs/mcp**：`src/rds-control-plane-mcp-server` 在主仓 404，根 README 0 提及，server-cards.json 无条目；
- **未发 PyPI**：`awslabs.rds-control-plane-mcp-server` / `awslabs.rds-management-mcp-server` / `awslabs.rds-monitoring-mcp-server` 均 404；DockerHub 公开镜像亦未能验证到；
- **组织整体停更**：三个仓库最后活动停在 2025-07-22 ~ 2025-08-06；
- 结论：它要么是在 Agent Toolkit 转型期被搁置，要么未来以 Agent Toolkit 形态重新发布。**今天是"未 GA 的团队仓库"状态**——要用只能 clone 源码跑，生产接入前 pin commit、自审代码、配最小 IAM。它的设计文档（RFC #617）和代码仍是理解"控制面 MCP 该怎么做"的最佳样本。

## 三、托管路线：Bedrock AgentCore 栈

AgentCore 是 AWS 的 agent 平台（2025 re:Invent 发布，现已含 12 个模块服务），与"数据库 MCP"直接相关的有五个：

- **Gateway（核心）**：全托管 AI 网关，一个入口承接 agent 流量。三种工具来源：OpenAPI 规范、Smithy 模型、Lambda 函数——几行代码转成 MCP 兼容工具；同时支持透传已有 MCP server 和 A2A 流量、跨模型 provider 的推理路由。关键能力：**入站认证 + 出站认证双向管理**（OAuth 流程、token 刷新、凭证保管，"唯一同时覆盖两者"的全托管服务）、语义工具选择（上千工具时压缩 prompt）、一键集成 Salesforce/Slack/Jira 等。对数据库场景的含义：你的内网数据库 API 不需要写 MCP server，用 OpenAPI 描述交给 Gateway 即可变成 MCP 工具。
- **Registry**：组织内 agent、MCP server、工具、skill 的目录，带发布/审核/批准工作流和语义搜索——解决"团队里 50 个 MCP server 谁先批谁后用"。
- **Policy**：用自然语言或 Dogwood 策略语言（兼容 Cedar）写规则，在 Gateway 上**拦截每一次工具调用**——哪个 agent 能调哪个工具、什么条件下可执行。这是托管路线对"数据面安全"的正式回答。
- **Identity**：agent 身份与认证（Cognito/Okta/Entra ID/Auth0），替代手工分发凭证。
- **Payments**：通过 x402 / MPP 协议给付费 MCP server 和内容做微支付——数据库 API 商业化计费的管道，值得留意。

另有 **AWS MCP Server（2026-05-06 GA）**：AWS 官方托管的远程 MCP server，2025-11 preview、2026-05-06 正式 GA——GA 版支持 **IAM context keys**（不再需要为它单独配置 IAM 权限，可用条件键做细粒度管控），预置 Agent SOPs（按 AWS 最佳实践的多步任务流程）、语法校验的 API 调用、零凭证暴露、CloudTrail 全审计。这是"托管路线"在官方服务本身的样板。**AWS Knowledge MCP Server** 是第二个托管 remote（文档与知识）。注意区分：AWS MCP Server 走"广覆盖 API + 托管"，aws-rds-mcp 走"RDS 专科 + 源码分发"，二者设计取向不同。

## 四、三条线对比

| 维度 | awslabs/mcp（开源） | aws-rds-mcp（开源，团队仓） | AgentCore 托管 |
|---|---|---|---|
| 运行位置 | 用户本机 / 自行部署 | 用户本机（源码运行） | AWS 全托管 serverless |
| 传输 | 仅 stdio（迁 Streamable HTTP 中） | 仅 stdio（uv run / docker） | Streamable HTTP 端点 |
| 凭证 | 本机 AWS 凭证 / secret_arn | 本机 AWS 凭证 | 零凭证暴露，IAM + Identity |
| 审计 | 无（本机进程） | 无（本机进程） | CloudTrail / CloudWatch 全链路 |
| 数据面保护 | `--readonly` 开关（自律） | 默认只读 + 破坏操作确认 token（自律） | Policy 规则（他律，逐调用拦截） |
| 覆盖面 | 20+ 数据库数据面一等公民 | 仅 RDS 控制面 + 监控诊断 | 任意 OpenAPI/Lambda 后端，数据库需自行包装 |
| 分发 | uvx / GitHub / Docker | **仅 GitHub 源码** | Registry 治理 + 语义发现 |
| 成熟度 | 活跃维护，9.8k stars | 2025-08 停更，未 GA，未入 PyPI | AWS MCP Server 已 GA；Gateway 持续扩模块 |
| 成本 | 免费（开源） | 免费（开源） | 按量计费 |

判断：三条线不是替代关系而是分工——awslabs/mcp 是"数据面参考实现 + 开发体验入口"，aws-rds-mcp 是"控制面专科实现 + 尚未转正"，Gateway 是"生产化的共同答案"。真正的张力在**数据面**：开源侧把写能力以 `--readonly` 开关交给你自己把关，托管侧用 Policy 把把关产品化。对合规团队，这不是选择题。

## 五、风险与注意事项

1. **MCP 通用风险在本场景放大**：协议层面的已知攻击（工具描述投毒、rug pull——恶意 server 在工具列表落地后替换实现）对数据库 server 尤其危险——一个被污染的 Redshift server 等于数据仓库凭据。pin 版本、审查 description、生产只接内部包装的 server。
2. **开源路线的凭证断层**：stdio server 用你本机的凭证执行，公司审计视角里这些操作"不是人就是 agent"，Agent Toolkit 的 IAM 条件键正是补这个洞——过渡期建议显式配置。
3. **写能力的默认开放问题**：多数数据面 server 未默认只读，agent 拿到 write 工具后的破坏半径取决于你的 IAM 粒度。最小权限原则：给 server 专用 IAM 角色，只授目标库必要动作。rds-management 这类控制面 server 的破坏半径是整库删实例，比数据面更危险，`--readonly` 应视为必选项而非可选项。
4. **团队仓的供应链风险**：aws-rds-mcp 三个仓库未入 PyPI、无签名发布物、2025-08 起停更——引用它等于把一段未经 AWS 发行流程审视的代码接进你的凭证边界。用可以，pin commit + 代码评审 + 独立 IAM 角色，三件套缺一不可。
5. **托管路线的锁定**：Gateway 的 OpenAPI 转换是好用，但工具目录、Policy、Identity 都在 AgentCore 内沉淀，跨云迁移成本随使用深度上升。
6. **文档口径漂移**：awslabs/mcp 根 README 与其 src README 之间存在已知不一致（MSSQL 认证方式、OpenSearch 去向）——引用任何一条前先落到 src README 核对；本文已按 src README 修正。

## 六、动手路径

1. **本机十分钟**：装好 `uv`，在 Claude Code / Cursor 配置里加一条 Aurora PostgreSQL server（`uvx awslabs.postgresql-mcp-server@latest --resource_arn ... --secret_arn ... --readonly true`），用自然语言查一次表结构。
2. **RDS 控制面体验**：clone `aws-rds-mcp/rds-management`，`uv --directory rds-management run main.py --readonly` 接进 MCP 客户端，让 agent 列一次集群/实例、读一份慢查询报告；想试合并版就 clone `aws-rds-mcp/rds-mcp` 跑 `src/rds-control-plane-mcp-server`。生产前置条件：pin commit、自审、专用 IAM 角色。
3. **团队共享**：把内部数据库维护团队的 Python 脚本用 FastMCP 包成只读 server 提交内审，或用 Gateway 把现有 OpenAPI 服务转 MCP——后者不用写 MCP 代码。控制面场景优先评估 AWS API MCP Server（广覆盖、elicitation 确认）与 rds-control-plane（专科、token 确认）哪个更适配你的客户端。
4. **生产化**：server 进 AgentCore Registry 走审批流；Gateway 前挂 Policy（先全只读试运行）；Identity 接 IdP；观测接 CloudWatch。写能力单独审批、单独角色、单独 Policy。
5. **迁移观察项**：盯 Agent Toolkit for AWS 的功能矩阵（aws-rds-mcp 的能力可能以它的形态重新出现）与 awslabs/mcp 的归档计划，新投入以 Agent Toolkit 为准。

## 附录

**A. 开源数据面 server 与访问方式速查**：RDS Data API（Aurora PG/MySQL，或 wire 直连）、Secrets Manager 取密直连（Oracle、MSSQL）、直连协议（DSQL、DocumentDB、Neptune、Keyspaces、Timestream、Valkey、Memcached）、AWS API（DynamoDB、S3 Tables、ElastiCache 控制面、SiteWise、Glue/EMR/Athena）、Lambda 包装（Lambda Tool）。OpenSearch 已移交社区项目 `opensearch-project/opensearch-mcp-server-py`。

**B. aws-rds-mcp 组织时间线**：2025-06-16 组织内首个仓库 rds-monitoring 创建；2025-06-18 awslabs/mcp RFC #617 发布；2025-06-20 rds-management 创建；2025-07-14 rds-mcp fork 创建（内含合并版 rds-control-plane-mcp-server）；2025-07-22 fork 最后 push；2025-07-29 rds-management 最后提交；2025-08-06 rds-monitoring 最后更新；截至 2026-10-19 未并入 awslabs/mcp、未上 PyPI。

**C. 关键日期**：2025-03-21 awslabs/mcp 创建；2025-05-26 SSE 支持移除；2025-11 AWS MCP Server preview；2025 re:Invent AgentCore 发布；2026-05 Agent Toolkit for AWS 上线；2026-05-06 AWS MCP Server GA（IAM context keys）；2026-10-19 awslabs/mcp 约 9.8k stars。

**D. 与 MCP 协议演进的咬合**：awslabs/mcp 从 SSE 撤向 Streamable HTTP，与协议 2026-07-28 版的无状态核心同向；托管 Gateway 天然适配无状态 per-request 模型。

## 参考资料

- [GitHub: awslabs/mcp（开源 server 全清单、stdio 传输说明、SSE 移除公告、Agent Toolkit 继任者提示，2026-10-19 抓取）](https://github.com/awslabs/mcp)
- [GitHub: awslabs/mcp src/mssql-mcp-server（认证方式以 src README 为准，2026-10-19 核对）](https://github.com/awslabs/mcp/tree/main/src/mssql-mcp-server)
- [GitHub: aws-rds-mcp/rds-management（RDS 控制面 CRUD，22 工具，2026-10-19 抓取）](https://github.com/aws-rds-mcp/rds-management)
- [GitHub: aws-rds-mcp/rds-monitoring（RDS 只读监控，6 工具，2026-10-19 抓取）](https://github.com/aws-rds-mcp/rds-monitoring)
- [GitHub: aws-rds-mcp/rds-mcp（awslabs/mcp fork，内含合并版 rds-control-plane-mcp-server，2026-10-19 核对）](https://github.com/aws-rds-mcp/rds-mcp)
- [RFC #617: RDS Control Plane MCP Server（设计文档：确认 token、最佳实践默认值、out of scope，2026-10-19 抓取）](https://github.com/awslabs/mcp/issues/617)
- [AWS What's New: AWS MCP Server 正式 GA（2026-05-06，IAM context keys）](https://aws.amazon.com/about-aws/whats-new/2026/05/aws-mcp-server/)
- [AWS 文档：What is Amazon Bedrock AgentCore（12 个模块服务全景：Gateway/Registry/Policy/Identity/Payments 等，2026-10 抓取）](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/what-is-bedrock-agentcore.html)
- [AWS 文档：AgentCore Gateway（OpenAPI/Smithy/Lambda 转 MCP、双向认证、语义工具选择、目标类型，2026-10 抓取）](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway.html)
- [GitHub API: awslabs/mcp 仓库元数据（9.8k stars、2025-03-21 创建、Apache-2.0、最近 push 2026-10-16，2026-10-19 查询）](https://api.github.com/repos/awslabs/mcp)
