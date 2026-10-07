# AWS 数据库 MCP 与 Skill 全景调研

> 类型 · 定位 · 使用场景 · 功能点 · 实现原理

AWS 在 Agent 时代的数据库工具体系是云厂商里跑得最完整的：一个统一仓库下 20+ 个数据库 MCP Server，配套三层 Skill 体系（MCP 内嵌 / agent-plugins / agent-toolkit），外加 Hooks 做写操作拦截。本文按五个维度拆解。

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
│  通过标准协议暴露 tools                      │
│  实际执行查询、Schema 变更、元数据读取         │
├─────────────────────────────────────────────┤
│  Plugins（打包单元）                         │
│  Skill + MCP + Hooks 的组合发行              │
│  一条命令装齐一类工作负载所需的全部能力        │
└─────────────────────────────────────────────┘
```

业界一个流行的概括：**Skills 是大脑，MCP 是手**。Skill 负责"这件事该怎么做"（业务逻辑、决策流程），MCP 负责"实际去执行"（带安全边界和审计）。AWS 是这套分工的典型案例。

---

## 三、类型盘点：数据库 MCP 矩阵

全部开源在 `github.com/awslabs/mcp`，按数据模型分类：

**关系型（5）**

| Server | 定位 | 执行通道 |
|---|---|---|
| Aurora PostgreSQL | NL2SQL + 元数据 | RDS Data API |
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

值得注意的是选型覆盖逻辑：AWS 数据库产品线有 15+ 引擎，MCP 不是每个服务一个，而是**每个开发者会动手操作的服务一个**，建模指导类（DynamoDB）与操作类分离。

---

## 四、三个功能范式

从功能设计看，AWS 数据库 MCP 明显走了三条不同的路。

### 4.1 执行型：Aurora MySQL/PostgreSQL——NL2SQL 管道

最典型。工具集：

- `sql_list_tables` / `sql_get_schema`：发现表与表结构
- `nl2sql`：自然语言转 SQL（内置方言规则 + LLM provider 抽象，支持 Amazon Bedrock 或 LiteLLM）
- `sql_run_query`：执行并返回结果
- `business_concepts` / `business_concepts_load` / `reset_context`：加载业务概念文件（自然语言术语映射），管理 schema 上下文

典型管道：发现表 → 选 schema 进上下文 → 生成 SQL → 执行 → 解释结果。上下文管理是显式工具（`reset_context`），说明设计者清楚 schema 挤占上下文窗口的问题。

### 4.2 指导型：DynamoDB——prompt-as-tool

2.0 版是一个重要转向：把"操作"（CRUD）剥离给通用的 AWS API MCP Server，自己只做**设计指导与数据建模**。核心工具：

- `dynamodb_data_modeling`：不改 LLM 上下文，检索并返回建模专家 prompt
- `source_db_analyzer`：从 MySQL schema + Performance Schema 提取访问模式，生成 DynamoDB 设计建议

这本质是**把专家知识以 prompt 形式作为工具返回**——rule-based，不需要 LLM 调用，且天然可组合（任何 LLM 都能用）。专门解决"关系型思维迁移到 NoSQL"这个最贵的问题。

### 4.3 极简型：Aurora DSQL——三工具原则

只有 `get_schema` / `readonly_query` / `transact`。但配套的 Skill 极重：DDL 迁移（表重建模式）、OCC 重试、多租户隔离、查询计划诊断，全部在 SKILL.md 的 reference 文件里按需加载。**工具面最小化，知识面最大化**——执行通道窄而安全，知识通过 Skill 注入。

---

## 五、实现原理

### 5.1 技术栈与分发

- 统一用 Python + FastMCP 框架实现
- 两种分发：`uvx`（`uv tool run`，零安装）和 Docker 镜像（发布到 ECR Public Gallery: `gallery.ecr.aws/awslabs-mcp/`）
- 传输：stdio（本地默认）与 HTTP/SSE（远程）；官方明确 SSE 正在移除，转向 streamable HTTP

### 5.2 安全模型（值得单独看）

AWS 数据库 MCP 的安全设计是一条完整链路：

1. **凭据不进上下文**：数据库密码存 Secrets Manager，MCP Server 通过 IAM 角色访问，LLM 永远看不到密钥
2. **RDS Data API 作为执行通道**：HTTP 接口，不直接连数据库端口，天然适合从容器/CI 环境调用，HTTP 响应天然适配 LLM 的上下文格式
3. **只读默认**：`--readonly` 启动参数、`readonly_query` 结构化只读工具
4. **写前校验**：DSQL 的 `dsql_lint` 在 DDL 执行前做静态检查
5. **Hooks 拦截**：agent-plugins 的 `databases-on-aws` 插件注册了 schema 验证 hook——`transact` 写操作执行后，自动提示验证 schema 变更与影响行数
6. **审计**：与 AWS API MCP Server 的 denyList / elicitList 策略配合（如 `aws rds delete-db-instance` 必须人工确认）

### 5.3 通用 AWS API MCP Server

2025 年后推出的 `aws-api-mcp-server`：把 AWS CLI 命令封装为 MCP 工具，支持只读模式、沙箱执行、CloudTrail 审计。DynamoDB 的表管理操作就迁到了这里。定位是"万能兜底"——具体服务的 MCP 管深度，它管广度。

---

## 六、Skill 体系：三种形态

AWS 的 Skill 有三个存放位置，对应三种用途。

### 6.1 嵌入 MCP Server 内部

如 `aurora-dsql-mcp-server/skills/`，与工具同仓发行。技能直接指导如何使用本 Server 的工具。

### 6.2 agent-plugins 仓库：插件化打包

`awslabs/agent-plugins` 的 `databases-on-aws` 插件 = Skill（DSQL 全部最佳实践）+ MCP Server（awsknowledge 文档检索 + aurora-dsql 操作，后者默认禁用）+ Hooks。一条 `/plugin install` 装齐。

### 6.3 agent-toolkit-for-aws：路由中枢

最新的 `aws/agent-toolkit-for-aws` 仓库里，`aws-database` 核心 Skill 是入口：

- **路由模式**：description 里写死"STOP——不要凭训练知识回答"，先把用户意图匹配到子技能注册表（15+ 引擎），再加载对应 service skill
- **知识卡片**：services.json 提供各服务的快速事实（版本、限制、GA 状态），保证信息新鲜度
- **artifact 交接**：选库 Skill 产出 `requirements.json`（引擎、区域、容量信号），下游 Skill（如 aurora-postgresql）读取后继续，避免用户重复输入
- **service skill 内部再路由**：如 Aurora PG Skill 有 20+ 子技能（express 配置、ACU 容量、pgvector、升级规划），匹配后只加载对应 reference 文件

这是**渐进式披露（progressive disclosure）**的完整实现：frontmatter 触发 → 路由 → 按需读 reference → 执行。Agent 上下文里始终只有当前任务需要的知识。

---

## 七、使用场景

AWS 官方博客给出四个，加上生态里的两个：

1. **Schema 驱动的特性开发**：AI 助手读取实时 schema 理解表关系，生成 CRUD 代码，随 schema 演进同步更新
2. **数据探索与业务洞察**：分钟级构建 dashboard（自动处理数据关联与可视化建议）
3. **测试代码生成**：基于 live schema 和访问模式生成针对性测试（约束验证、DynamoDB 访问模式、缓存 TTL 场景）
4. **监控与排障**：自然语言查询缓存内存、复制状态、慢查询——AI 摘要替代人肉解析 INFO 输出
5. **异构迁移**：MySQL → DynamoDB 的建模转换（source_db_analyzer）
6. **ChatBI / 数据民主化**：Redshift 只读查询 + 元数据浏览，非技术人员直接问数

集成面：Amazon Q CLI / Q Developer、Cursor、VS Code、Claude Desktop、Windsurf、Kiro 等；服务端有 Bedrock AgentCore 托管运行时。

---

## 八、观察与启示

站在数据库产品视角，几个值得注意的判断：

**1. 工具面做窄，知识面做宽。** DSQL 只有 3 个工具但 Skill 极重，DynamoDB 只做设计指导。Agent 时代的工具设计不是"暴露全部 API"，而是"暴露最小安全操作面 + 注入最大决策知识"。

**2. 安全是分层工程，不是一个开关。** 凭据隔离（Secrets Manager）→ 执行通道隔离（Data API）→ 只读默认 → 写前 lint → 写后 hook → CloudTrail 审计。每一层都由不同组件承担。

**3. Skill 与 MCP 解耦但协同。** Skill 是可读、可版本控制的 Markdown 知识包（产品经理可直接参与维护），MCP 是带安全边界的执行通道。两者通过插件机制打包发行。

**4. "不要凭训练知识回答"是产品决策。** aws-database Skill 强制路由到子技能，本质是把模型的"记忆"替换为"检索"——保证答案时效性与可审计。这与 Agent Harness 博客里"context engineering > prompt engineering"的判断一致。

**5. 对竞争的意味：** AWS 把 15+ 数据库引擎逐一 MCP 化，是在把"数据库产品"变成"Agent 的原生能力"。数据库厂商之间的下一个竞争维度，可能是谁的元数据接口、文档体系、Skill 包对 Agent 最友好。

---

## 参考资料

1. AWS Database Blog: Supercharging AWS database development with AWS MCP servers (2025-06) — https://aws.amazon.com/blogs/database/supercharging-aws-database-development-with-aws-mcp-servers/
2. GitHub: awslabs/mcp（官方 MCP Server 合集仓库）— https://github.com/awslabs/mcp
3. GitHub: awslabs/agent-plugins（databases-on-aws 插件）— https://github.com/awslabs/agent-plugins
4. GitHub: aws/agent-toolkit-for-aws（aws-database 核心 Skill 与数据库技能集）— https://github.com/aws/agent-toolkit-for-aws
5. aurora-dsql-mcp-server SKILL.md（渐进式披露结构范例）— https://github.com/awslabs/mcp/blob/main/src/aurora-dsql-mcp-server/skills/aws-dsql-skill/SKILL.md
6. ECR Public Gallery: awslabs-mcp（容器分发）— https://gallery.ecr.aws/awslabs-mcp/awslabs/postgres-mcp-server
7. dev.to: Build Faster with Amazon Q Developer — MCP vs Agent Skills — https://dev.to/jackohhearts/build-faster-with-amazon-q-developer-mcp-vs-agent-skills-and-aws-cost-dashboards-1pdc
8. Medium: Building a Complete Amazon Aurora MySQL MCP Server — https://medium.com/@michaelwpace/building-a-complete-amazon-aurora-mysql-mcp-server-a-comprehensive-guide-77f5e8eba4aa
9. GitHub: awslabs/mcp — aws-api-mcp-server（安全策略设计）— https://github.com/awslabs/mcp/tree/main/src/aws-api-mcp-server
