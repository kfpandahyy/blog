# 羲和 HaloDB 调研：功能全景、优势拆解与主打场景

> 一句话结论：HaloDB 是杭州易景数通基于 PostgreSQL 内核的国产商业数据库，主打"一个内核、三种兼容模式（Oracle/MySQL/PG）"的协议级迁移替代，功能纵深集中在兼容层（语法/函数/视图/系统包/过程语言五层清单），配合 MGR 分组复制等高可用组件，主攻信创场景的存量数据库平替。同类对比中，它与瀚高 IvorySQL 路线最近，差异化在多模式合一与税务、金融的头部落地。

**TL;DR**
- 底座 PG 深度 fork，实例级参数 `database_compat_mode` 切换 oracle/mysql/postgresql 三种模式；参数枚举里还留了 **sybase、sqlserver** 占位——路线图信号
- 兼容层有五层实物：语法（Oracle 27+ 项、MySQL 15+ 项清单见正文）、函数、数据字典视图（10 个 Oracle 视图）、系统包（9 个 DBMS_* / UTL_* 包）、过程语言（专用扩展 `plorasql` 跑 PL/SQL）
- 高可用组件四件套：MGR 分组复制、GDD 跨地域、DLB 负载均衡、HDS 分片，宣称 RPO=0 / RTO<30s
- 优势是结构性的：协议级兼容（驱动零替换）、PG 生态插件红利、运维命令零学习成本；短板是分布式组件无公开细节、性能宣称缺独立基准、公司信息不透明
- 主打场景很明确：税务/金融/政务的 Oracle 与 MySQL 存量迁移，不是互联网高并发弹性场景

## 一、产品速览

| 项 | 内容 |
|---|---|
| 出品 | 杭州易景数通科技有限公司（Halo Tech Co., Ltd.），halodbtech.com |
| 底座 | PostgreSQL 深度 fork（Halo14，实测安装包 `psql 1.0.14.12`），官网已预告 v2.0"内生智能" |
| 形态 | 闭源商业包（标准/企业/专业/开发版），license.lic 绑定硬件指纹（MAC/IP/CPU 序列号/主板序列号），可设有效期与最大并发数 |
| 协议 | PG 5432/1921 + MySQL 3306 双监听，可伪装版本应答（如 `8.0.21-log`） |
| 开源面 | openHalo（MySQL 兼容层，GPL-3.0，415 stars）+ HMT 迁移工具 |
| 资质 | 2026-09-20 通过安全可靠测评（2026 年第 3 号公告）；工信部信创产品图谱；麒麟/统信/鲲鹏/飞腾/海光等互认证 |
| 标杆 | 国家税务总局某核心系统：158 个复杂场景零修改通过（官方口径） |

注意与 yahoo/HaloDB（2018 年开源的 Java 嵌入式 KV 引擎，已停更）同名异物，本文不讨论。

## 二、功能全景梳理

### 2.1 多模式兼容引擎（核心资产）

三层机制：双监听（协议层）→ 二次解析 `standard_parserengine_auxiliary`（编译层旁路，不重写 PG 主解析器）→ 扩展加载（执行层，`create extension aux_oracle/aux_mysql cascade`）。

**Oracle 模式兼容清单**（据 2024-05 社区实测文章逐条验证，部分列表）：

- 语法层 27 项：连接符 `||`、带别名 UPDATE、匿名子查询、窗口函数 DISTINCT、DUAL 伪表、SYSDATE/SYSTIMESTAMP、ROWNUM、ROWID、MINUS 集合运算、DECODE（不限参数）、DBLINK、Oracle 数值格式输出、VARCHAR2/NUMBER/CLOB/BLOB 类型族、START WITH/CONNECT BY 层次查询、IN 特殊语法、Oracle 日期类型及运算、LISTAGG/WM_CONCAT、HAVING 与 GROUP BY 任意顺序、NULL 与空串等价、Oracle 序列取值语法、ORDER BY 用列别名、TABLE 函数、MERGE INTO、WITHIN GROUP、Q' 转义、NLS_DATE_FORMAT 参数、Oracle 格式 PROCEDURE/FUNCTION/PACKAGE 创建语法
- 数据字典视图 10 个：DBA_SEGMENTS、PRODUCT_COMPONENT_VERSION、USER_CON_COLUMNS、USER_CONSTRAINTS、USER_IND_COLUMNS、USER_OBJECTS、USER_PROCEDURES、USER_SOURCE、USER_TAB_COLUMNS、USER_TABLES
- 系统包 9 个：DBMS_ALERT、DBMS_ASSERT、DBMS_OBFUSCATION_TOOLKIT、DBMS_OUTPUT、DBMS_PIPE、DBMS_RANDOM、DBMS_UTILITY、UTL_FILE、UTL_I18N
- 过程语言：**plorasql** 扩展（"Oracle Compatible SQL (PL/SQL) procedural language"）——这是 Oracle 迁移最关键的一层，绝大多数"PG 系兼容库"死在存储过程上，HaloDB 单独做了过程语言扩展
- 扩展组合实测：aux_init + aux_oracle(1.8) + plorasql + plpgsql + pgcrypto

**MySQL 模式兼容清单**：MySQL 协议直连（mysql CLI 即可登录）、基本数据类型、引号规则、REPLACE INTO、INSERT IGNORE、ON DUPLICATE KEY UPDATE、INSERT..VALUE、LIMIT 家族（含 UPDATE/DELETE..ORDER BY..LIMIT）、ORDER BY 指定列序、SET/ENUM 类型、用户变量。配套语义映射：MySQL 的 database = halo0root 下的 schema；`password_encryption='mysql_native_password'`；用户分两套——MySQL 用户只能走 3306 且官方建议仅用于 DML，权限仍收在 PG role 体系。

**模式与路线图**：`database_compat_mode` 枚举实测含 postgresql/oracle/mysql/**sybase/sqlserver**（后两者尚未实现，见 2026-05 墨天轮体验文章），即团队规划是"一套框架吃遍主流商业库迁移"。

### 2.2 性能特性（官方口径，未见独立基准）

复杂 SQL 优化、过程语言（PL/SQL）执行优化、智能缓存、向量化执行，宣称综合性能比同类产品高 50%+。官网配置样例出现 `halodb.auto_index`（自动索引）、`halodb.ai_optimizer`（AI 优化器）——v2.0"内生智能"的方向，AI 与查询优化融合。

### 2.3 高可用与分布式

| 组件 | 官方功能 | 可得信息 |
|---|---|---|
| MGR 分组复制 | 同步分组复制，异地容灾 | 实现 RPO=0 的主体；细节未公开 |
| GDD | 跨地域分布部署 | 多机房能力，细节未公开 |
| DLB | 负载均衡 | 连接调度 |
| HDS | 数据分片 | 水平扩展；分片事务/一致性协议无公开资料 |

宣称 RPO=0、RTO<30s。注意这四件只在商业版；开源 openHalo 不含。

### 2.4 安全与运维

安全模块提供认证/授权/用户管理；官网持续发布关键补丁公告（2023-12、2024-06、2025-04），修复项清一色是上游 PG CVE 跟随（CVE-2024-4317、CVE-2024-0985、CVE-2025-1094、CVE-2024-10976、pg_signal_backend 等）——既是 rebase 机制运转的证据，也意味着**安全水位线 = PG 社区水位线**。运维面刻意对齐 PG：pg_ctl/initdb/psql/diag 日志全套沿用，默认库 halo0root，学习成本近零。

### 2.5 工具链与生态

- HMT：开源迁移工具（`HaloLab001/hmt-web`），自动化 schema + data 迁移
- openHalo：MySQL 兼容层开源版，415 stars，2026-07 仍活跃，Discord 社区、北大研究生课程入驻
- PG 插件生态：内核是 PG，PostGIS 等社区插件理论上可直接用（社区实测文章明确点出这是相对自研内核库的潜在优势）
- 安装包自带 Oracle instantclient（`product/instantclient_12_2`），兼容层与官方客户端互验

## 三、优势拆解（结构性，不是宣传语）

1. **协议级兼容 vs 语法级兼容**。多数国产库的"兼容"止于 SQL 方言翻译，应用驱动、连接池、监控探针都要换。HaloDB 让 Oracle OCI/JDBC、MySQL 驱动直接连，迁移改造成本从"应用层"压到"配置层"——这是"98% 代码减少"宣称的结构性来源，可信度高于同类宣传。
2. **一个内核三种模式，实例级切换**。同一集群可按库选择模式，迁移期新旧应用混跑、`database_compat_mode` 切回验证，回滚成本低。对比"一类负载一套专用库"的方案，运维面收敛。
3. **PL/SQL 过程语言是深水区能力**。`plorasql` 独立扩展 + 9 个 DBMS/UTL 系统包 + 10 个数据字典视图，覆盖的是金融核心库存量过程代码——这类代码往往占迁移工作量的大头，也是"零修改"案例（税务局 158 场景）能成立的技术前提。
4. **PG 生态红利**。深度 fork 而非自研：内核持续吸收社区特性，插件体系可扩展，DBA 人才市场现成，CVE 跟随修复。代价是每次 PG 大版本 rebase 的兼容层维护成本。
5. **信创资质完整**。安全可靠测评（2026-09）+ 信创图谱 + 全栈互认证，政企采购的硬性门槛已清。

## 四、主打场景

**场景一：Oracle 存量平替（核心场景）**。税务、金融、政务核心系统。适配逻辑：PL/SQL 存量代码靠 plorasql + 系统包；应用连接靠协议兼容；跑在飞腾/鲲鹏上还有国产芯片加成（官方称飞腾平台部分场景 2-3 倍于 Oracle）。典型案例即国家税务总局。这是 HaloDB 资源的绝对重心。

**场景二：MySQL 上信创**。openHalo 是这条线的先锋：互联网公司 MySQL 存量在信创要求下的低成本替代，复用 MySQL 驱动与 DBA 习惯。

**场景三：多类型数据库整合（"全场景统一数据库"叙事）**。用一套产品承接 Oracle + MySQL + PG 多种存量，消除数据孤岛、收敛运维。这是官网定位层的表述，落地深度取决于 HDS 分片的成熟度，公开信息最少，需 POC 验证。

**非目标场景**：互联网高并发弹性伸缩（无云服务能力证据）、极致 OLAP（无列存/向量化深度公开资料）、全球分布式（GDD 细节不明）。选型时别被"全场景"三个字带偏。

## 五、宣称 vs 可验证事实

| 宣称 | 可验证性 | 评估 |
|---|---|---|
| 代码修改减少 98%+ | 仅官方案例 | 方向可信（协议级兼容的结构优势），数字待考 |
| 80% 场景性能优于 Oracle（同 Intel 平台） | 仅官方 | 需同机 POC；复杂 PL/SQL 重负载是关键变量 |
| RPO=0 / RTO<30s | 仅官方 | 同步复制可实现 RPO=0；RTO 需故障注入实测 |
| MySQL 模式比 MySQL 更快（openHalo） | 有学生基准项目 | 参考意义大于结论意义 |
| MySQL 5.7+ 低修改迁移 | 社区多篇文章实测支持 | 常用语法覆盖属实；存储过程/驱动特殊行为仍是风险区 |

## 六、同类坐标

| 产品 | 路线 | 与 HaloDB 的关系 |
|---|---|---|
| 瀚高 IvorySQL | PG + Oracle 兼容，Apache 2.0 开源（2021 起） | 最接近的同类；更早、社区更深，但只做 Oracle 单模式 |
| 崖山 YashanDB | 自研内核 + Oracle 兼容 | 兼容层同源思路，内核路线不同 |
| 达梦/金仓 | 自研内核 + 迁移工具链 | 非 PG 生态，兼容靠改造 |
| OceanBase | 分布式自研 + Oracle/MySQL 双模式 | 分布式强，但架构重、非 PG 系 |
| AWS Babelfish | PG + SQL Server 协议/方言 | 国际先例，证明该路线工程可行 |

## 七、判断与建议

1. HaloDB 的产品定义非常聚焦：**卖给"必须换掉 Oracle/MySQL 且不想改代码"的机构**，而不是"想要更好数据库"的人。这个约束下，"PG 内核 + 协议壳"比自研内核风险低，plorasql 是它区别于普通兼容库的关键资产。
2. 风险集中在三点：公司体量与融资不透明；HDS/GDD 分布式组件零公开细节；v2.0 切换期兼容承诺需合同锁定。
3. POC 建议按真实存量压测：导出你们自己的 PL/SQL 包和存储过程集跑一遍 plorasql，再测 MGR 故障切换 RTO——这两项过了，98% 和 30 秒才是你的数字，否则只是官网的数字。

## 参考资料

- [HaloDB 官网（2026-10-20 抓取）：E5 引擎、MGR/GDD/DLB/HDS、税务局案例、安全可靠测评](https://www.halodbtech.com/)
- [GitHub: HaloTech-Co-Ltd/openHalo（GPL-3.0，415 stars，2026-10-20 核对）](https://github.com/HaloTech-Co-Ltd/openHalo)
- [ITPUB：HaloDB 的 Oracle 兼容模式（语法/视图/系统包清单实测，2024-05）](https://blog.itpub.net/70041262/viewspace-3030930/)
- [OSCHINA：HaloDB 的 MySQL 兼容模式（协议与语法清单，2024-05）](https://my.oschina.net/mysqloffice/blog/16533603)
- [墨天轮：从 HaloDB 体验到国产数据库兼容性（三种模式实测 + sybase/sqlserver 参数占位发现，2026-05）](https://www.modb.pro/db/1908850441186717696)
- [墨天轮：国产数据库 HaloDB 从小白到专家（安装、license、版本实操，2024-09）](https://www.modb.pro/db/1838235854394978304)
- [GitHub: yahoo/HaloDB（同名异物，已停更）](https://github.com/yahoo/HaloDB)

---

*数据截至 2026-10-20。性能与兼容度数字除标注外均为官方口径。*
