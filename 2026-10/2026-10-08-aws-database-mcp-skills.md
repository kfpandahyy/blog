# AWS 数据库 MCP 与 Skill 全景调研

> 类型 · 定位 · 使用场景 · 功能点 · 实现原理

AWS 在 Agent 时代的数据库工具体系是云厂商里跑得最完整的：管理面与数据面 MCP 分治，一个统一仓库下 20+ 个数据库 MCP Server，配套三层 Skill 体系（MCP 内嵌 / agent-plugins / agent-toolkit），外加 Hooks 做写操作拦截。本文按五个维度拆解。

---

## 一、问题背景

传统数据库开发的痛点，AWS 官方博客总结得很准：开发者在 IDE、psql/mysql 客户端、文档之间反复切换，同时要维护多套数据模型的心智（SQL 方言差异、关系型 vs NoSQL 的建模范式差异）。MCP 的价值主张：把数据库的元数据与操作能力，以标准协议注入 AI 编码助手的工作现场，让模型"看见"真实的 schema、访问模式与运行状态，而不是凭训练记忆猜。

---

## 二、整体布局：三层架构

AWS 的玩法不是一堆散装 MCP Server，而是一个组合体系：

```
┌─────────────────────────────────────────────┐
│  Skills（脑）                                │
│  SKILL.md 格式的领域知识包                   │
│  决策、路由、最佳实践、工作流                  │
├─────────────────────────────────────────────┤
│  MCP Servers（手）                           │
│  管理面：实例/集群生命周期（控制面）          │
│  数据面：查询、Schema、NL2SQL（数据面）       │
├─────────────────────────────────────────────┤
│  Plugins（打包单元）                         │
│  Skill + MCP + Hooks 的组合发行              │
│  一条命令装齐一类工作负载所需的全部能力        │
└─────────────────────────────────────────────┘
```

业界一个流行的概括：**Skills 是大脑，MCP 是手**。Skill 负责"这件事该怎么做"（业务逻辑、决策流程），MCP 负责"实际去执行"（带安全边界和审计）。AWS 是这套分工的典型案例。

---

## 三、核心分野：管理面 MCP vs 数据面 MCP

这是理解 AWS 数据库 MCP 体系的第一把钥匙。**控制面操作和数据面操作被拆成不同的 Server**，各自独立仓库、独立权限模型。

### 3.1 管理面 MCP：RDS Management MCP Server

- **仓库**：`github.com/aws-rds-mcp/rds-management`（独立仓库，不在 awslabs/mcp 下）
- **定位**：数据库资源生命周期管理——对应 AWS 控制台 / RDS API 的那层能力
- **工具面**：
  - 集群管理：CreateDBCluster / ModifyDBCluster / DeleteDBCluster / ChangeDBClusterStatus（启停重启）/ FailoverDBCluster（强制故障转移）
  - 快照与恢复：CreateDBClusterSnapshot / DeleteDBClusterSnapshot / RestoreDBClusterFromSnapshot / RestoreDBClusterToPointInTime
  - 实例管理：CreateDBInstance / ModifyDBInstance / DeleteDBInstance / ManageDBInstanceStatus
  - 参数组管理：集群与实例参数组的创建、修改、重置、查询
- **资源模板**：`aws-rds://db-cluster`、`aws-rds://db-instance`（列表 + 单实例详情），让 Agent 先"看见"资源再操作
- **安全**：`--readonly` 启动参数屏蔽一切变更操作；官方建议给 LLM 单独配只读 IAM 角色，与人权限分离

典型场景：用自然语言让 Agent "把 dev 环境的 MySQL 8.0 实例从 db.r6g.large 改到 db.r6g.xlarge，然后打个快照"——整个变更链路由模型编排，人只审批。

### 3.2 数据面 MCP：以 Aurora PostgreSQL 为代表

- **仓库**：`awslabs/mcp` 下的 `src/postgres-mcp-server`（MySQL 对应 `src/mysql-mcp-server`）
- **定位**：数据操作与查询——对应 psql/mysql 客户端的那层能力
- **工具面**：
  - `sql_list_tables` / `sql_get_schema`：发现表与表结构
  - `nl2sql`：自然语言转 SQL（内置方言规则 + LLM provider 抽象，支持 Amazon Bedrock 或 LiteLLM）
  - `sql_run_query`：执行并返回结果
  - `business_concepts` / `business_concepts_load` / `reset_context`：加载业务概念文件（自然语言术语映射），管理 schema 上下文
- **执行通道**：RDS Data API（HTTP，凭据走 Secrets Manager，不直连数据库端口）；新版也支持直连连接串（`--connection-string`），自托管 PostgreSQL 也能用

典型管道：发现表 → 选 schema 进上下文 → 生成 SQL → 执行 → 解释结果。上下文管理是显式工具（`reset_context`），说明设计者清楚 schema 挤占上下文窗口的问题。

### 3.3 为什么要拆

管理面与数据面的拆分不是技术洁癖，是三个现实约束的结果：

1. **权限模型不同**：管理操作走 IAM + RDS API（控制面凭据），数据操作走数据库账号（数据面凭据）。合在一起意味着一个 Server 要同时持有两种高价值凭据。
2. **风险等级不同**：DeleteDBInstance 是分钟级不可逆的资损操作，DELETE FROM 是数据操作（有备份可救）。分开后可以对管理面默认只读、数据面放开写，策略粒度更细。
3. **变更频率不同**：数据面工具随方言和场景快速演进（NL2SQL、业务概念层），管理面工具跟随 RDS API 版本走，迭代节奏完全不同。

---

## 四、数据面 MCP 的全景矩阵

数据面 MCP 全部在 `awslabs/mcp` 仓库，按数据模型分类：

**关系型（5）**

| Server | 定位 | 执行通道 |
|---|---|---|
| Aurora PostgreSQL | NL2SQL + 元数据 | RDS Data API / 直连 |
| Aurora MySQL | NL2SQL + 元数据 | RDS Data API |
| Aurora DSQL | 分布式 SQL，三工具极简设计 | 直连 + IAM 认证 |
| RDS Oracle | Oracle 操作 | Secrets Manager 认证 |
| RDS MSSQL | SQL Server 操作 | RDS Data API |

**NoSQL（5）**

| Server | 定位 |
|---|---|
| DynamoDB | 设计指导与数据建模（2.0 后操作移走） |
| DocumentDB | MongoDB 兼容文档操作 |
| Neptune | 图查询（openCypher / Gremlin） |
| Keyspaces | Cassandra 兼容宽列操作 |
| Timestream for InfluxDB | 时序数据操作 |

**分析与搜索（3）**

| Server | 定位 |
|---|---|
| Redshift | 数仓元数据浏览 + 只读查询 |
| S3 Tables | S3 上的分析表管理 |
| OpenSearch | 搜索与分析 |

**缓存与内存（3）**：ElastiCache Valkey / Memcached、MemoryDB Valkey——偏运维诊断场景（内存用量、复制状态、慢查询排查）。

值得注意的是选型覆盖逻辑：AWS 数据库产品线有 15+ 引擎，数据面 MCP 不是每个服务一个，而是**每个开发者会动手操作的服务一个**，建模指导类（DynamoDB）与操作类分离。

---

## 五、数据面 MCP 的三个功能范式

从功能设计看，数据面 MCP 明显走了三条不同的路。

### 5.1 执行型：Aurora MySQL/PostgreSQL——NL2SQL 管道

上节已述。这是数据面 MCP 的主流派：发现元数据 → 生成 SQL → 执行 → 解释。

### 5.2 指导型：DynamoDB——prompt-as-tool

2.0 版是一个重要转向：把"操作"（CRUD）剥离给通用的 AWS API MCP Server，自己只做**设计指导与数据建模**。核心工具：

- `dynamodb_data_modeling`：不改 LLM 上下文，检索并返回建模专家 prompt
- `source_db_analyzer`：从 MySQL schema + Performance Schema 提取访问模式，生成 DynamoDB 设计建议

这本质是**把专家知识以 prompt 形式作为工具返回**——rule-based，不需要 LLM 调用，且天然可组合（任何 LLM 都能用）。专门解决"关系型思维迁移到 NoSQL"这个最贵的问题。

### 5.3 极简型：Aurora DSQL——三工具原则

只有 `get_schema` / `readonly_query` / `transact`。但配套的 Skill 极重：DDL 迁移（表重建模式）、OCC 重试、多租户隔离、查询计划诊断，全部在 SKILL.md 的 reference 文件里按需加载。**工具面最小化，知识面最大化**——执行通道窄而安全，知识通过 Skill 注入。

---

## 六、实现原理

### 6.1 技术栈与分发

- 统一用 Python + FastMCP 框架实现
- 两种分发：`uvx`（`uv tool run`，零安装）和 Docker 镜像（发布到 ECR Public Gallery: `gallery.ecr.aws/awslabs-mcp/`）
- 传输：stdio（本地默认）与 HTTP/SSE（远程）；官方明确 SSE 正在移除，转向 streamable HTTP

### 6.2 安全模型（值得单独看）

AWS 数据库 MCP 的安全设计是一条完整链路，且管理面与数据面各有侧重：

**数据面**：

1. **凭据不进上下文**：数据库密码存 Secrets Manager，MCP Server 通过 IAM 角色访问，LLM 永远看不到密钥
2. **RDS Data API 作为执行通道**：HTTP 接口，不直接连数据库端口，天然适合从容器/CI 环境调用
3. **语义级只读**：postgres-mcp-server 用 `pglast`（PostgreSQL 官方解析器 libpg_query）解析每条 SQL，只放行 SELECT 类语句形状；且能识别"伪装成 SELECT 的写操作"——`nextval()` 改序列状态、`pg_stat_reset()` 清统计、`pg_switch_wal()` 切日志，一律拒绝
4. **最小权限角色的双保险**：`--privilege_check` 参数在连接时校验数据库角色——超级用户、`rds_superuser` 成员、BYPASSRLS 角色直接拒绝连接（`enforce` 模式）；推荐用专用只读角色连接，即使 SQL 过滤被绕过，数据库层权限仍是最后边界
5. **写前校验**：DSQL 的 `dsql_lint` 在 DDL 执行前做静态检查
6. **Hooks 拦截**：agent-plugins 的 `databases-on-aws` 插件注册了 schema 验证 hook——`transact` 写操作执行后，自动提示验证 schema 变更与影响行数

**管理面**：

- `--readonly` 整体屏蔽变更工具
- 官方明确建议：给 LLM 的 IAM 角色与人的角色分开，只授所需权限（如只授 Describe*）
- 高危操作（DeleteDBCluster 等）在 AWS API MCP Server 层还有 denyList / elicitList 二次确认

### 6.3 通用执行层：AWS API MCP Server 与托管版 AWS MCP Server

- **aws-api-mcp-server**（开源）：把 AWS CLI 命令封装为 MCP 工具，支持只读模式、沙箱执行、CloudTrail 审计。DynamoDB 的表管理操作就迁到了这里。定位是"万能兜底"——具体服务的 MCP 管深度，它管广度。
- **AWS MCP Server（托管版，2026 年 GA）**：AWS 把散装的开源 Server 整合成一个全托管端点（目前 us-east-1 / eu-central-1），能力 = 文档检索 + AWS API 调用 + 脚本执行 + 官方 Skills 四合一。安全上引入 **IAM 上下文键（context keys）区分人类与 Agent 的身份**，所有调用留 CloudTrail + CloudWatch 审计。Server 本身免费，只对创建的资源计费。对企业来说，这意味着从"自己运维一堆本地 MCP Server"切换到"接一个端点"。

---

## 七、Skill 体系：三种形态

AWS 的 Skill 有三个存放位置，对应三种用途。

### 7.1 嵌入 MCP Server 内部

如 `aurora-dsql-mcp-server/skills/`，与工具同仓发行。技能直接指导如何使用本 Server 的工具。

### 7.2 agent-plugins 仓库：插件化打包

`awslabs/agent-plugins` 的 `databases-on-aws` 插件 = Skill（DSQL 全部最佳实践）+ MCP Server（awsknowledge 文档检索 + aurora-dsql 操作，后者默认禁用）+ Hooks。一条 `/plugin install` 装齐。

### 7.3 agent-toolkit-for-aws：路由中枢

最新的 `aws/agent-toolkit-for-aws` 仓库里，`aws-database` 核心 Skill 是入口：

- **路由模式**：description 里写死"STOP——不要凭训练知识回答"，先把用户意图匹配到子技能注册表（15+ 引擎），再加载对应 service skill
- **知识卡片**：services.json 提供各服务的快速事实（版本、限制、GA 状态），保证信息新鲜度
- **artifact 交接**：选库 Skill 产出 `requirements.json`（引擎、区域、容量信号），下游 Skill（如 aurora-postgresql）读取后继续，避免用户重复输入
- **service skill 内部再路由**：如 Aurora PG Skill 有 20+ 子技能（express 配置、ACU 容量、pgvector、升级规划），匹配后只加载对应 reference 文件

这是**渐进式披露（progressive disclosure）**的完整实现：frontmatter 触发 → 路由 → 按需读 reference → 执行。Agent 上下文里始终只有当前任务需要的知识。

---

## 八、使用场景

AWS 官方博客给出四个，加上生态里的两个，以及管理面 MCP 带来的新场景：

1. **Schema 驱动的特性开发**：AI 助手读取实时 schema 理解表关系，生成 CRUD 代码，随 schema 演进同步更新
2. **数据探索与业务洞察**：分钟级构建 dashboard（自动处理数据关联与可视化建议）
3. **测试代码生成**：基于 live schema 和访问模式生成针对性测试（约束验证、DynamoDB 访问模式、缓存 TTL 场景）
4. **监控与排障**：自然语言查询缓存内存、复制状态、慢查询——AI 摘要替代人肉解析 INFO 输出
5. **异构迁移**：MySQL → DynamoDB 的建模转换（source_db_analyzer）
6. **ChatBI / 数据民主化**：Redshift 只读查询 + 元数据浏览，非技术人员直接问数
7. **自然语言运维（管理面独有）**：实例规格变更、快照管理、故障转移演练——用对话完成原本在控制台里的操作，全程 CloudTrail 留痕

集成面：Amazon Q CLI / Q Developer、Cursor、VS Code、Claude Desktop、Windsurf、Kiro 等；服务端有 Bedrock AgentCore 与托管 AWS MCP Server。

---

## 九、观察与启示

站在数据库产品视角，几个值得注意的判断：

**1. 管理面与数据面分治是 Agent 时代的刚需。** 不是因为拆起来漂亮，而是权限模型、风险等级、迭代节奏三者都不同。做数据库 MCP 的产品，第一步就该想清楚这层拆分，而不是先做一个"大而全"的 Server 再事后补权限。

**2. 工具面做窄，知识面做宽。** DSQL 只有 3 个工具但 Skill 极重，DynamoDB 只做设计指导。Agent 时代的工具设计不是"暴露全部 API"，而是"暴露最小安全操作面 + 注入最大决策知识"。

**3. 安全是分层工程，不是一个开关。** 凭据隔离（Secrets Manager）→ 执行通道隔离（Data API）→ 语义级只读（pglast 解析 SQL，识别伪装的写操作）→ 角色权限双保险（privilege_check）→ 写前 lint → 写后 hook → CloudTrail 审计。每一层都由不同组件承担。

**4. "不要凭训练知识回答"是产品决策。** aws-database Skill 强制路由到子技能，本质是把模型的"记忆"替换为"检索"——保证答案时效性与可审计。这与 Agent Harness 博客里"context engineering > prompt engineering"的判断一致。

**5. 从开源散装到托管整合是确定的演进方向。** AWS 的路线：先开源一堆单点 MCP Server（2025）→ 再出通用 API Server 兜底（2025 末）→ 最后 GA 托管端点 + IAM 身份区分 + 全量审计（2026）。对数据库厂商的启示：MCP Server 的终局可能不是"用户部署我的 Server"，而是"我把能力接进一个托管层"。

**6. 对竞争的意味：** AWS 把 15+ 数据库引擎逐一 MCP 化，又补上管理面，是在把"数据库产品"变成"Agent 的原生能力"。数据库厂商之间的下一个竞争维度，可能是谁的元数据接口、文档体系、Skill 包、管理面 MCP 对 Agent 最友好。

---

## 参考资料

1. AWS Database Blog: Supercharging AWS database development with AWS MCP servers (2025-06) — https://aws.amazon.com/blogs/database/supercharging-aws-database-development-with-aws-mcp-servers/
2. GitHub: awslabs/mcp（数据面 MCP Server 合集仓库）— https://github.com/awslabs/mcp
3. GitHub: aws-rds-mcp/rds-management（管理面 MCP Server）— https://github.com/aws-rds-mcp/rds-management
4. GitHub: awslabs/agent-plugins（databases-on-aws 插件）— https://github.com/awslabs/agent-plugins
5. GitHub: aws/agent-toolkit-for-aws（aws-database 核心 Skill）— https://github.com/aws/agent-toolkit-for-aws
6. aurora-dsql-mcp-server SKILL.md（渐进式披露结构范例）— https://github.com/awslabs/mcp/blob/main/src/aurora-dsql-mcp-server/skills/aws-dsql-skill/SKILL.md
7. awslabs.postgres-mcp-server PyPI（pglast 语义只读与 privilege_check 设计）— https://pypi.org/project/awslabs.postgres-mcp-server/
8. ECR Public Gallery: awslabs-mcp（容器分发）— https://gallery.ecr.aws/awslabs-mcp/awslabs/postgres-mcp-server
9. serverworks blog: AWS MCP Server GA 移行记（托管版能力对比，2026-05）— https://blog.serverworks.co.jp/aws-mcp-server-ga-2026
10. dev.to: Build Faster with Amazon Q Developer — MCP vs Agent Skills — https://dev.to/jackohhearts/build-faster-with-amazon-q-developer-mcp-vs-agent-skills-and-aws-cost-dashboards-1pdc
11. Medium: Building a Complete Amazon Aurora MySQL MCP Server — https://medium.com/@michaelwpace/building-a-complete-amazon-aurora-mysql-mcp-server-a-comprehensive-guide-77f5e8eba4aa
12. GitHub: awslabs/mcp — aws-api-mcp-server（安全策略设计）— https://github.com/awslabs/mcp/tree/main/src/aws-api-mcp-server
