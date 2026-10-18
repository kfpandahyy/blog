# AWS 数据库 MCP 全景：开源工具箱与托管网关的两条路线

> **TL;DR**：AWS 的数据库 MCP 版图分两层——开源侧是 `awslabs/mcp` 仓库（9.8k stars，2025-03 创建，Apache-2.0）里 20+ 个仅支持 stdio 的本地 server，覆盖 DynamoDB、Aurora（PG/MySQL/DSQL）、Redshift、DocumentDB、Neptune、Keyspaces、Timestream、Oracle/MSSQL、Valkey/Memcached、S3 Tables、OpenSearch 及 Glue/EMR/Athena 数据管道，分发方式是 uvx + FastMCP，认证走 RDS Data API 或 Secrets Manager；托管侧是 Bedrock AgentCore 的 Gateway（把 OpenAPI/Smithy/Lambda 转成 MCP 工具的双向认证网关）+ Registry + Policy，以及 preview 阶段的官方托管 AWS MCP Server（IAM 权限、零凭证暴露、CloudTrail 全审计）。选型一句话：本地开发接开源 server，生产共享工具走 Gateway 托管，数据面操作务必开 `--readonly` 或套 Policy。
> **数据截至 2026-10-18。** 开源侧以 `awslabs/mcp` 仓库 README 为准，托管侧以 Bedrock AgentCore 官方文档为准。

---

你的二分法是对的，而且比表面更深一层：AWS 在数据库 MCP 这件事上同时跑着**两条产品路线**——一条把 20 多个数据库的 MCP server 当开源软件发（本地 stdio、自己跑、自己管凭证），一条把 MCP 当托管云服务卖（AgentCore Gateway 托管端点、按量计费、IAM 集成）。前者解决"我的开发机上 agent 能不能用数据库"，后者解决"全公司的 agent 怎么安全地共享同一批工具"。两条路线的技术取舍几乎处处相反。

本文的路线：先盘点开源 server 清单与架构，再拆托管 AgentCore 栈，然后对比两条路线的安全与运维模型，最后给选型和动手路径。

一句话心智模型：**开源路线是"把数据库的钥匙交给本机的 agent"，托管路线是"把数据库放在 agent 够得着但有门卫的柜台后面"。**

---

## 一、定义与边界

- **调研对象**：AWS 官方（awslabs / AWS 服务团队）发布的数据库相关 MCP server 与托管 MCP 能力；不含社区第三方 server，不含非数据库类（仅在与数据库场景交叉时提及）。
- **"管理服务 vs 数据服务"的口径**：管理服务 = 控制面操作（建表、改配置、扩节点、查监控）；数据服务 = 数据面操作（查询、读写、执行 SQL）。这个二分贯穿全文，因为两条路线在两者上的取舍不同。
- **信息源**：一手（GitHub 仓库、AWS 官方文档）为主；所有数字带日期。

## 二、开源路线：awslabs/mcp 仓库解剖

### 2.1 基本面

- 仓库 `awslabs/mcp`，2025-03-21 创建，Apache-2.0，截至 2026-10 约 9.8k stars / 1.8k forks，是 AWS Labs 增长最快的项目之一。
- **传输：仅 stdio**。2025-05-26 起全部移除 SSE 支持，正在向 Streamable HTTP 迁移——与 MCP 协议本身的演进节奏一致（协议 2026-07 版核心已转向无状态、每请求自携带协商信息）。
- **技术栈**：Python + FastMCP 框架，通过 `uvx` 一键运行，例如 `uvx awslabs.mysql-mcp-server@latest`。Cursor、VS Code、Kiro、Claude Code、Windsurf 都有对应的一键安装配置。
- **官方已发出迁移信号**：README 顶部公告称 "Agent Toolkit for AWS 已上线，是 MCP servers、plugins、skills 的继任者"——增加了 IAM 条件键（区分 agent 操作与人类操作）、CloudWatch/CloudTrail 可见性、经过准确性评估的 skills。awslabs/mcp 继续维护但新重心在 Agent Toolkit。

### 2.2 数据库 server 清单（按"管理服务/数据服务"重排）

**数据服务（数据面，20 个）**：

| Server | 数据访问方式 | 备注 |
|---|---|---|
| Aurora PostgreSQL | RDS Data API | resource_arn + secret_arn |
| Aurora MySQL | RDS Data API | 同上 |
| Aurora DSQL | 直连 | 分布式 SQL，PG 兼容 |
| RDS Oracle | Secrets Manager 认证 | 经 Secrets Manager 取凭证 |
| RDS MSSQL | RDS Data API | SQL Server |
| DynamoDB | AWS API | 两个版本：设计指导/数据建模版 + 完整表管理版 |
| Redshift | SQL 查询 | 数仓操作与分析查询 |
| DocumentDB | MongoDB 兼容协议 | 文档操作 |
| Neptune | openCypher / Gremlin | 图查询 |
| Keyspaces | Cassandra 兼容 | 宽列操作 |
| Timestream (InfluxDB) | InfluxDB 兼容 | 时序 |
| S3 Tables | AWS API | 面向分析的表存储 |
| OpenSearch | OpenSearch API | 搜索/分析/可观测 |
| ElastiCache/MemoryDB for Valkey | Valkey 协议 | 高级数据结构 |
| ElastiCache for Memcached | Memcached 协议 | 缓存读写 |
| AWS Data Processing | Glue / EMR-EC2 / Athena | 管道可见性 + 执行 |
| IoT SiteWise | SiteWise API | 工业 IoT 数据 |
| Bedrock Knowledge Bases | 检索 API | 带引用的知识查询 |
| Kendra / Q Index | 检索 API | 企业搜索 |
| Lambda Tool | 执行 Lambda | 把私有资源访问包装成工具 |

**管理服务（控制面）**：ElastiCache（完整控制面操作）、S3 Tables 管理、DynamoDB 表管理（完整版内含）、以及外围的 IAM、CloudWatch、Billing、Pricing、CloudTrail 等运维 server。

### 2.3 架构与设计决策

- **认证模式是这组 server 最重要的工程决策**：Aurora 系一律走 **RDS Data API**——HTTPS 接口、免直连数据库、天然利用 RDS 的 IAM 集成，凭证只出现在 Data API 的 secret_arn 引用里，不进连接串。Oracle 走 Secrets Manager。这把"agent 拿数据库凭证"这个最大攻击面压缩到了 IAM 边界内。
- **只读开关**：数据库类 server 普遍提供 `--readonly` 参数。面向 agent 暴露写能力前，先问自己能否接受模型误删。
- **工具描述即接口文档**：每个 server 的 description 写得详细（DynamoDB 甚至分出"设计指导"专用 server），因为模型选工具完全依赖它——这也是 MCP 工具投毒攻击面在本场景的直接映射：从仓库安装的 server 应 pin 版本。
- **本地运行的边界**：stdio 意味着 server 跑在用户机器上，用的是用户的 AWS 凭证，出网流量也过本机——方便开发与调试，但凭证暴露面与审计断层是生产不能接受的直接原因。

## 三、托管路线：Bedrock AgentCore 栈

AgentCore 是 AWS 的 agent 平台（2025 re:Invent 发布，现已含 12 个模块服务），与"数据库 MCP"直接相关的有五个：

- **Gateway（核心）**：全托管 AI 网关，一个入口承接 agent 流量。三种工具来源：OpenAPI 规范、Smithy 模型、Lambda 函数——几行代码转成 MCP 兼容工具；同时支持透传已有 MCP server 和 A2A 流量、跨模型 provider 的推理路由。关键能力：**入站认证 + 出站认证双向管理**（OAuth 流程、token 刷新、凭证保管，"唯一同时覆盖两者"的全托管服务）、语义工具选择（上千工具时压缩 prompt）、一键集成 Salesforce/Slack/Jira 等。对数据库场景的含义：你的内网数据库 API 不需要写 MCP server，用 OpenAPI 描述交给 Gateway 即可变成 MCP 工具。
- **Registry**：组织内 agent、MCP server、工具、skill 的目录，带发布/审核/批准工作流和语义搜索——解决"团队里 50 个 MCP server 谁先批谁后用"。
- **Policy**：用自然语言或 Dogwood 策略语言（兼容 Cedar）写规则，在 Gateway 上**拦截每一次工具调用**——哪个 agent 能调哪个工具、什么条件下可执行。这是托管路线对"数据面安全"的正式回答。
- **Identity**：agent 身份与认证（Cognito/Okta/Entra ID/Auth0），替代手工分发凭证。
- **Payments**：通过 x402 / MPP 协议给付费 MCP server 和内容做微支付——数据库 API 商业化计费的管道，值得留意。

另有 **AWS MCP Server（preview）**：AWS 官方托管的远程 MCP server，预置 Agent SOPs（按 AWS 最佳实践的多步任务流程）、语法校验的 API 调用、IAM 权限、零凭证暴露、CloudTrail 全审计——这是"托管路线"在官方服务本身的样板。AWS Knowledge MCP Server 是第二个托管 remote（文档与知识）。

## 四、两条路线对比

| 维度 | 开源 awslabs/mcp | AgentCore 托管 |
|---|---|---|
| 运行位置 | 用户本机 / 自行部署 | AWS 全托管 serverless |
| 传输 | 仅 stdio（迁 Streamable HTTP 中） | Streamable HTTP 端点 |
| 凭证 | 本机 AWS 凭证 / secret_arn | 零凭证暴露，IAM + Identity |
| 审计 | 无（本机进程） | CloudTrail / CloudWatch 全链路 |
| 数据面保护 | `--readonly` 开关（自律） | Policy 规则（他律，逐调用拦截） |
| 覆盖面 | 20+ 数据库一等公民 | 任意 OpenAPI/Lambda 后端，数据库需自行包装 |
| 分发 | uvx / GitHub | Registry 治理 + 语义发现 |
| 成本 | 免费（开源） | 按量计费 |
| 生态信号 | 维护中，官方重心迁向 Agent Toolkit | AgentCore 持续扩模块（已 12 个） |

判断：两条路线不是替代关系而是分工——开源 server 是"AWS 数据库团队的 MCP 参考实现 + 开发体验入口"，Gateway 是"生产化的共同答案"。真正的张力在**数据面**：开源侧目前把写能力以 `--readonly` 开关交给你自己把关，托管侧用 Policy 把把关产品化。对合规团队，这不是选择题。

## 五、风险与注意事项

1. **MCP 通用风险在本场景放大**：协议层面的已知攻击（工具描述投毒、rug pull——恶意 server 在工具列表落地后替换实现）对数据库 server 尤其危险——一个被污染的 Redshift server 等于数据仓库凭据。pin 版本、审查 description、生产只接内部包装的 server。
2. **开源路线的凭证断层**：stdio server 用你本机的凭证执行，公司审计视角里这些操作"不是人就是 agent"，Agent Toolkit 的 IAM 条件键正是补这个洞——过渡期建议显式配置。
3. **写能力的默认开放问题**：多数数据面 server 未默认只读，agent 拿到 write 工具后的破坏半径取决于你的 IAM 粒度。最小权限原则：给 server 专用 IAM 角色，只授目标库必要动作。
4. **托管路线的锁定**：Gateway 的 OpenAPI 转换是好用，但工具目录、Policy、Identity 都在 AgentCore 内沉淀，跨云迁移成本随使用深度上升。
5. **预览状态**：AWS MCP Server 仍是 preview；Agent Toolkit 与 awslabs/mcp 的迁移路径未完全清晰——不要在 Q4 规划里默认它们已 GA。

## 六、动手路径

1. **本机十分钟**：装好 `uv`，在 Claude Code / Cursor 配置里加一条 Aurora PostgreSQL server（`uvx awslabs.postgresql-mcp-server@latest --resource_arn ... --secret_arn ... --readonly true`），用自然语言查一次表结构。
2. **团队共享**：把内部数据库维护团队的 Python 脚本用 FastMCP 包成只读 server 提交内审，或用 Gateway 把现有 OpenAPI 服务转 MCP——后者不用写 MCP 代码。
3. **生产化**：server 进 AgentCore Registry 走审批流；Gateway 前挂 Policy（先全只读试运行）；Identity 接 IdP；观测接 CloudWatch。写能力单独审批、单独角色、单独 Policy。
4. **迁移观察项**：盯 Agent Toolkit for AWS 的 GA 公告与 awslabs/mcp 的归档计划，新投入以 Agent Toolkit 为准。

## 附录

**A. 开源数据面 server 与访问方式速查**：RDS Data API（Aurora PG/MySQL、MSSQL）、Secrets Manager（Oracle）、直连协议（DSQL、DocumentDB、Neptune、Keyspaces、Timestream、Valkey、Memcached、OpenSearch）、AWS API（DynamoDB、S3 Tables、ElastiCache 控制面、SiteWise、Glue/EMR/Athena）、Lambda 包装（Lambda Tool）。

**B. 关键日期**：2025-03-21 awslabs/mcp 创建；2025-05-26 SSE 支持移除；2025-09-08 官方 MCP Registry 预览；2025 re:Invent AgentCore 发布；2026 期间 Agent Toolkit for AWS 上线（继任者信号）；2026-10 awslabs/mcp 约 9.8k stars。

**C. 与 MCP 协议演进的咬合**：awslabs/mcp 从 SSE 撤向 Streamable HTTP，与协议 2026-07-28 版的无状态核心同向；托管 Gateway 天然适配无状态 per-request 模型。

## 参考资料

- [GitHub: awslabs/mcp（开源 server 全清单、stdio 传输说明、SSE 移除公告、Agent Toolkit 继任者提示，2026-10 抓取）](https://github.com/awslabs/mcp)
- [AWS 文档：What is Amazon Bedrock AgentCore（12 个模块服务全景：Gateway/Registry/Policy/Identity/Payments 等，2026-10 抓取）](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/what-is-bedrock-agentcore.html)
- [AWS 文档：AgentCore Gateway（OpenAPI/Smithy/Lambda 转 MCP、双向认证、语义工具选择、目标类型，2026-10 抓取）](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/gateway.html)
- [GitHub API: awslabs/mcp 仓库元数据（9.8k stars、2025-03-21 创建、Apache-2.0，2026-10-18 查询）](https://api.github.com/repos/awslabs/mcp)
