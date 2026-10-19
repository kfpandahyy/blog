# 羲和 HaloDB 调研：给 PostgreSQL 装上 Oracle/MySQL 的"协议外壳"

> 一句话结论：HaloDB（羲和数据库）是杭州易景数通基于 PostgreSQL 内核做的国产商业数据库，核心卖点是用"协议级兼容"承接 Oracle/MySQL 存量应用的信创迁移；其兼容层以开源项目 openHalo 的形式部分放出。技术路线与瀚高 IvorySQL 同类，差异在"一套内核、三种协议、按配置切换"的产品化包装。

**TL;DR**
- HaloDB = PG 内核 + E5/MMC 多模式兼容引擎（Oracle/MySQL/PG 三种兼容模式，配置切换）+ MGR 分组复制/GDD/DLB/HDS 高可用组件，商业 license 授权（绑定硬件指纹）
- 开源面：**openHalo**（`HaloTech-Co-Ltd/openHalo`，GPL-3.0，C 语言，415 stars，2025-03 创建，2026-07 仍活跃）——PG 直接监听 MySQL 协议（3306），`CREATE EXTENSION aux_mysql` 即启用
- 已实现路径很清晰：PG fork → 双监听（5432 + 3306）→ MySQL 方言解析（二次解析）→ 类型/系统目录/函数映射 → `mysql_native_password` 认证
- 关键判断：**兼容是真的，但"零修改迁移"需按场景打折**——MySQL 模式语义差异（db=schema、用户体系分离）意味着复杂应用仍要测试；性能宣称（比 Oracle 优、比 MySQL 快）目前只有官方案例，缺独立基准
- 注意与 yahoo/HaloDB（2018 年开源的 Java 嵌入式 KV 引擎）完全是两个东西，同名而已

## 一、先分清两个 HaloDB

这个名字有歧义，搜索时容易串：

| | 羲和 HaloDB（本文） | yahoo/HaloDB |
|---|---|---|
| 出品 | 杭州易景数通科技 | Yahoo/Oath（Arjun Mannaly） |
| 定位 | 国产商业关系型数据库（信创） | Java 嵌入式 KV 存储引擎 |
| 底座 | PostgreSQL 内核 | 自研 LSM 变体（内存哈希索引 + append-only log） |
| 协议 | PG/Oracle/MySQL 多协议 | 嵌入式库，无协议 |
| 状态 | 商业在售，v2.0 预告中 | 2018 年开源，528 stars，2023 年后停更 |
| 官网 | halodbtech.com / openhalo.org | github.com/yahoo/HaloDB |

下文只讨论前者。

## 二、公司与产品定位

**公司**：杭州易景数通科技有限公司（Halo Tech Co., Ltd.），国家高新技术企业、工信部信创工委会会员单位、数据库标准工作组成员单位。融资与规模信息未公开。

**资质（截至 2026-10-20 官网口径）**：
- 羲和通用数据库系统软件 V1.0 于 2026-09-20 通过中国信息安全测评中心与国家保密科技测评中心的"安全可靠测评"（2026 年第 3 号公告）——这是信创采购的硬门槛
- 入选工信部信创产品图谱；与麒麟、统信、中科方德、东方通、宝兰德、鲲鹏、飞腾、兆芯、海光等完成互认证
- 版本体系：Halo14（1.0.14.x，2024 年安装包实测 `psql (1.0.14.12)`）为当前主售版本，官网已挂"Halo Database v2.0 即将发布"；分标准版/企业版/专业版/开发版

**目标客户与打法**：金融、税务、政务等信创替代场景。标杆案例是**国家税务总局某核心业务系统**：宣称 158 个复杂场景全部通过、应用代码零修改、同 Intel 平台 80% 场景性能优于 Oracle、飞腾平台部分场景 2-3 倍。这是典型的"Oracle 平替"叙事，同赛道对手是达梦、人大金仓、OceanBase、TiDB、PolarDB、YashanDB 以及同样是"PG 套兼容壳"的瀚高 IvorySQL。

## 三、技术架构：兼容引擎怎么实现的

官方话术是"E5 引擎 + MMC 多模式兼容"，拆解到可验证的层面，实现路径是标准的 PG 深度 fork 四件套：

**1. 双监听（协议适配层）**
- 主端口走 PG 协议（默认 1921），MySQL 模式开启 `second_listener_on` + `second_port = 3306`（openHalo 里参数名是 `mysql.listener_on` / `mysql.port`）
- MySQL 客户端可直连，且能伪装版本应答：`mysql.halo_mysql_version = '8.0.21-log'`，连 `SHOW VARIABLES` 类的兼容都做了
- 字符序支持 `mysql.ci_collation`（大小写不敏感）

**2. 方言解析（编译层）**
- 参数 `database_compat_mode = 'postgresql' | 'mysql' | 'oracle'`，实例级切换三种模式
- `standard_parserengine_auxiliary = 'on'` 即"二次解析"：PG 原生解析器走不通的方言语句，进入辅助解析引擎再翻译一次。这是兼容层的关键设计——不重写主解析器，而是挂一个旁路
- MySQL 模式的能力清单（2024-05 社区实测）：REPLACE INTO、INSERT IGNORE、ON DUPLICATE KEY UPDATE、LIMIT 语法族、SET/ENUM 类型、用户变量等；Oracle 模式覆盖 date 类型映射（`oracle.use_datetime_as_date`）、`transform_null_equals`（`=` 判 NULL）等语义差异点

**3. 系统目录与对象映射（执行层）**
- MySQL 模式下"database 即 schema"：MySQL 客户端看到的每个 db，实际是 `halo0root` 库下的一个 schema——这是 PG 内核做多协议的天然映射方案（AntDB、TBase 早期也这么干）
- `CREATE EXTENSION aux_mysql` / `aux_oracle`（cascade）：兼容能力以扩展形式加载，插件式内核架构的官方依据
- MySQL 用户与 Halo 用户分离：MySQL 用户只能走 3306 业务端口，且官方建议只用于 DML——权限模型仍然落在 PG 的 role 体系上

**4. 认证兼容**
- `set password_encryption = 'mysql_native_password'` 后可创建 MySQL 客户端可用的账号，密码存进 `pg_shadow`

这套打法的工程含义：**兼容性上限高**（贴着 PG 社区演进，内核本身持续吸收 PG 新特性），**代价是兼容层的长期维护成本**——PG 每次大版本升级，二次解析器和协议层都要跟一遍。官网安全公告（2024-06、2025-04 补丁，涉及 CVE-2024-4317、CVE-2024-0985、CVE-2025-1094 等）清一色是上游 PG CVE 的跟随修复，从侧面印证了"深度 fork + 跟随 rebase"的维护模式。

## 四、高可用与分布式组件

官方列的组件缩写，按 PG 生态常识对应：

| 组件 | 功能（官方口径） | 性质判断 |
|---|---|---|
| MGR 分组复制 | 同步分组复制，异地容灾，RPO=0 | 类似 PG 同步复制组 + 集群管理（非 MySQL MGR 技术） |
| GDD | 跨地域分布部署（Global Distributed Deployment） | 分布式/多机房能力，细节未公开 |
| DLB | 负载均衡 | 连接调度层 |
| HDS | 数据分片 | 水平扩展分片 |

容灾指标宣称 RPO=0、RTO<30s。注意：这些组件只在商业版提供，openHalo 开源版不含；且 HDS/GDD 的架构细节（一致性协议、跨片事务）公开资料几乎没有，POC 时需要逐项验证。

## 五、openHalo：开源的那一半

**基本信息（截至 2026-10-20）**：`HaloTech-Co-Ltd/openHalo`，GPL-3.0，C 语言，415 stars / 49 forks，2025-03-24 创建，最近 push 2026-07-24，官方站 openhalo.org，有 Discord 社区（宣称 50+ 活跃贡献者，入驻北大研究生课程）。

**它是什么**：把 HaloDB 的 MySQL 兼容层单独开源——让原生 PostgreSQL 获得 MySQL 协议与方言能力。README 原话："PostgreSQL to work with applications written for MySQL but provides much more better performance than MySQL"。

**启用路径**（开源版实测流程）：
```sql
-- postgresql.conf
database_compat_mode = 'mysql'
mysql.listener_on = true
mysql.port = 3306

-- SQL
CREATE EXTENSION aux_mysql CASCADE;
SET password_encryption = 'mysql_native_password';
CREATE USER test PASSWORD 'test';
```
然后 `mysql -P 3306 -h ...` 直连。

**生态配套**：迁移工具 HMT（`HaloLab001/hmt-web`，schema + data 自动化迁移，开源）；法国 IMT Atlantique 学生团队做过兼容性测试与性能基准项目（fxcollot/openHaloDocumentation）。

**定位**：openHalo 是 HaloDB 的技术名片和市场漏斗——GPL-3.0 对企业商用有传染性约束，真要上生产（尤其需要 Oracle 兼容、高可用组件时），唯一路径是买商业版。这是国产数据库常见的一手安排，对标瀚高的 IvorySQL 之于 HighGo DB。

## 六、部署形态与 license 模型

- **闭源商业包**：CentOS/RHEL 系 tarball（如 `halo_14.el7.x86_64`），目录结构 `product/{dbms, instantclient_12_2, shield}`（instantclient 目录直接带 Oracle 客户端——兼容层开发与 Oracle 官方客户端互验的痕迹；shield 疑为license/守护组件）
- **license.lic**：绑定 MAC、IP、CPU 序列号、主板序列号的硬件指纹授权，支持设置生效/失效时间、最大并发连接数；信息不匹配数据库拒绝启动
- 初始化、启停命令与 PG 完全一致（`pg_ctl`/`initdb`/`psql`），默认库 `halo0root`，diag 日志目录结构同 PG——运维面刻意零学习成本，这也是"迁移友好"的一部分
- 配置参数全是 PG 风格（shared_buffers、wal_level 等），另加 `halodb.*` 前缀私有参数（官网示例出现 `halodb.auto_index`、`halodb.ai_optimizer`——AI 优化器方向，v2.0 看点）

## 七、宣称 vs 可验证事实

| 宣称 | 可验证性 | 评估 |
|---|---|---|
| Oracle 兼容、代码修改减少 98%+ | 仅官方案例 | 税务总局案例无法独立复核；158 个场景是什么复杂度未知。**方向可信，数字待考** |
| 80% 场景性能优于 Oracle | 仅官方 | 需同机同数据集 POC；"优于 Oracle"在 PG 内核 OLTP 场景并不罕见，但复杂 PL/SQL 重负载是另一回事 |
| RPO=0、RTO<30s | 仅官方 | MGR 分组复制若为同步复制实现则 RPO=0 成立，RTO 数字需故障注入实测 |
| 比 MySQL 性能更好（openHalo） | 有第三方学生基准项目 | 复杂度有限，参考意义大于结论意义 |
| MySQL 5.7+ 应用低修改迁移 | 社区实测文章支持 | 常用语法覆盖属实；存储过程、特殊驱动行为仍属风险区 |

## 八、同类产品坐标

- **瀚高 IvorySQL**：同为"PG + Oracle 兼容"开源路线（Apache 2.0），与 openHalo 的"MySQL 兼容"方向互补；IvorySQL 更早（2021），社区根基更深
- **AntDB / TBase / PG 系国产库**：多走"PG 内核 + 分布式"，兼容层不是主卖点
- **达梦 / 金仓 / OceanBase**：自研或深度改造内核，兼容靠迁移工具链，路线不同
- **Babelfish (AWS)**：SQL Server 兼容 on PG 的国际先例——证明"PG 套商业协议外壳"路线在工程上成立，HaloDB 是同一思路的国产化 + 多协议化版本

## 九、判断

1. **技术路线成立且工程完成度不低**。双监听 + 二次解析 + 扩展化兼容层，是可维护性较好的兼容方案；安全公告跟随上游 PG CVE 也说明 rebase 机制在运转。
2. **价值集中在信创迁移窗口**。它的客户不是"想要更好的数据库"，而是"必须换掉 Oracle/MySQL 且不想改代码"的机构——在这个约束下，"PG 内核 + 多协议壳"比自研内核风险低得多。
3. **风险点**：公司体量与融资信息不透明；HDS/GDD 等分布式组件无公开架构细节；v2.0 临近，版本切换期的兼容性承诺需写进合同；GPL-3.0 的 openHalo 与企业商用之间有明确的付费墙。
4. **试用建议**：非信创场景，直接用 PG + IvorySQL/openHalo 拼更自由；信创场景，HaloDB 值得进 POC 名单，重点压测你们自己的 PL/SQL 或 MySQL 存储过程存量——那才是兼容层的真实水位线。

## 参考资料

- [HaloDB 官网（2026-10-20 抓取）：E5 引擎、MGR/GDD/DLB/HDS、税务总局案例、安全可靠测评公告](https://www.halodbtech.com/)
- [GitHub: HaloTech-Co-Ltd/openHalo（GPL-3.0，415 stars，2026-10-20 核对）](https://github.com/HaloTech-Co-Ltd/openHalo)
- [openHalo 官方站与安装/兼容模式教程](https://www.openhalo.org)
- [墨天轮：国产数据库 HaloDB 从小白到专家（安装、license、Oracle/MySQL 模式实操，2024-09）](https://www.modb.pro/db/1838235854394978304)
- [少安事务所系列：HaloDB Oracle/MySQL 兼容模式实测（2024-05）](https://segmentfault.com/a/1190000044928987)
- [GitHub: yahoo/HaloDB（同名异物的嵌入式 KV，2026-10-20 核对）](https://github.com/yahoo/HaloDB)

---

*数据截至 2026-10-20。文中性能与兼容宣称均为官方口径，已单独标注可验证性。*
