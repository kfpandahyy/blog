# DBAIOps 代码仓考古：cfe9d30，论文背后的 7.6 万行工程底座

> 仓库：github.com/OpenDataBox/DBAIOps，commit `cfe9d30`（2025-08-01，"init DBAIOps"，全仓唯一 commit）。
> 姊妹篇：[DBAIOps 论文精读：知识图谱能力如何被量化评测](./2026-10-09-dbaiops-paper.md)。上篇读论文说了什么，这篇读代码实际给了什么。

## 一句话

这个 commit 是 DBAIOps 投稿匿名评审期的完整开源版——7.6 万行 Shell/Python、664 张元数据表、60+ 数据库采集脚本、Neo4j 图谱构建工具链全部在；但**论文宣传的 LLM 诊断层一行代码都不在**。它是"底座全开源、大脑在服务端"的典型样本，也是读"学术论文 vs 开源代码"落差的最好教材。

## 一、先搞清楚版本关系：cfe9d30 ≠ master

今天该仓库的 master 分支是一个 Chrome 浏览器扩展（知识问答 / AWR 分析 / 巡检 / SQL 优化，对接云端服务），论文引用的 `HEval_criteria.md` 在其中 404。cfe9d30 则是被覆盖前的初版提交，内容完全不同。判断依据：

- README 底部链到 `anonymous.4open.science/r/DBAIOps-80F8`——匿名评审专用链接，说明这是**投稿版 artifact**；
- README 宣称"47.85% higher in diagnosis accuracy"，而 VLDB 正式版论文的数字是 34.85%（根因准确率）/ 47.22%（人工评估）——README 是投稿口径的化石；
- 附录 PDF 的标题抬头还是 LaTeX 模板占位符（"Conference'17, July 2017"）。

也就是说：**master 分支的 Chrome 扩展是发表后的"降级替换 artifact"，论文系统的真实代码底座只活在这个历史 commit 里**。用户指定这个 commit 号，指得极准。

## 二、系统全景：一个重装交付的私有化运维平台

这不是一个算法 demo，是一个可交付的**企业级软件安装包**：RHEL/CentOS/SUSE/麒麟 V10 一键安装（`bin/DBAIOps.sh -install`），单机 16C32G 起、60 实例以上推荐三台 16C64G，装完 1-1.5 小时，默认口令 admin/admin@123。技术栈从安装脚本可直接读出：

| 层 | 组件 | 代码证据 |
|---|---|---|
| 元数据库 | PostgreSQL（DSmart 元数据） | `pgconf/postgresql.conf`、`DBAIOps_2018.sql`（22 万行 DDL，664 张表） |
| 图谱库 | Neo4j（bolt 协议） | `bin/DBAIOps-neo4j.sh`、`update_knowl.py` |
| Web | Tomcat | `webserver/lib/apache-tomcat`（DBAIOps-web.sh 进程检查） |
| 报告渲染 | PhantomJS + OpenOffice + 中文字体 | `phantomjsconf/cn_fonts/simsun.ttc` |
| 缓存/协调 | Redis（含主从脚本）、ZooKeeper | `DBAIOps-redis*.sh`、`DBAIOps-zookeeper.sh` |
| 采集执行 | Python3 + cx_Oracle + psycopg2 + RSA 加密连接串 | `colscript/cib.py` |

目录结构就是架构图：`colscript/` 采集 → PG 元数据库（664 表：AWR 明细、告警规则、采集任务、健康检查评分……）→ Neo4j 图谱 → `knowl/` 报告生成 → Tomcat 前端。

## 三、分层解读：四件套各自的代码实证

### 3.1 采集层 colscript：论文"25 种数据库"的真实底座

37,759 行、60+ 脚本，覆盖清单比论文写的还野：

- **国外商业库**：Oracle、SQL Server、DB2
- **开源库**：MySQL、PostgreSQL、MongoDB、Redis、Kafka、HBase、HDFS、Flink
- **国产库几乎全阵容**：达梦 DM8、openGauss/GaussDB、GBase、GoldenDB、TiDB、TDSQL、OceanBase、崖山 Yashan、神通 Shentong、人大金仓 Kingbase
- **基础设施**：Linux/AIX/HP-UX 操作系统、Ceph、华为存储、Pacemaker

采集分三类，命名即方法论：**cib**（配置信息基线，静态配置采集）、**metric**（动态指标）、**health**（健康检查）。`cib.py` 是公共框架——连接池、RSA 解密数据库密码、超时装饰器、结果对象。论文里说"25 种数据库"，代码告诉你这不是论文修辞，是 60 个脚本一个一个适配出来的。

### 3.2 图谱层 update_knowl.py：Excel 到 Neo4j 的传送带

全文最重要的一个文件，论文"半自动构图"的工程实现：

1. **三类节点**，CQL 标签是中文的——`运维经验知识`、`指标知识`、`执行计划规则`；
2. **输入是 Excel 不是 MOS 文档**：`common_knowledge.xlsx` / `oracle_knowledge.xlsx` / `oracle_index.xlsx`，DBA 维护表格 → pandas 逐行 → 生成 `CREATE (n:...) ` CQL；
3. **知识内容 AES 加密入库**：`problems`、`special_rule` 字段过 `Mcrypt` 类（AES + 硬编码密钥）加密后存进 Neo4j——**图谱本身带防白嫖设计**，这也解释了为什么开源仓库里没有图谱数据；
4. **字段即论文 schema 的工程投影**：`relate_index` / `exclude_index`（相关/排除指标）对应论文 Relevance / Containment 边的属性化存储；`problem_index` / `desc_index` 标记知识是否索引化；
5. **图谱是平台可插拔组件**：`bin/DBAIOps_initmcs.py` 把 bolt 地址和 AES 密文密码写入元数据库 `sys_param`（code=995），分析服务名 `metric_kg_analyze`、按指标 ID 取图谱知识的函数名 `neo4j_knowl_by_indexid`——图谱检索在系统里是注册制的。

注意落差：论文描述的构图是"LLM 从 15,000 篇 MOS 文档抽取候选 + 多数投票 + DBA 校验"；工程版是"DBA 维护 Excel 直接导入"。LLM 抽取大概率是学术版后来加的能力（或服务端能力），**工程底座的知识生产方式是纯人工**。

### 3.3 知识层 knowl：2.8 万行"老 DBA 的毕生功力"

如果说图谱是骨架，这一层就是血肉——全部是规则化、模板化的专家经验：

- **CVE 漏洞核查**（`bugcheck/`）：`cve.xlsx` 存 Oracle/MySQL 各 CVE 的影响版本与补丁号，与采集到的实例版本/PSU/平台/RAC 做匹配，输出受影响清单。227 行，纯规则。
- **GUC 参数模板矩阵**（`conf/`）：这是全仓最见功力的资产——openGauss/GaussDB 的推荐参数，按 **硬件规格（4c16g 到 128c1024g 十几档）× 部署形态（集中式 / 云 / 华为云栈 hcs1/hcs2）** 组织成几十份 XML。白鳝身兼华为云 GaussDB HCDE，这层是他另一个身份的注脚。
- **巡检/月报/周报生成器**（`inspection_report/` 14 脚本、`monthly_report/` 30+ 脚本、`week_report/` 8 脚本）：章节化模板，1.1 基本信息、2.2 健康分析七个细目、6.x 日志分析、7 Top SQL、10 日志切换、13 问题检查——编号即 Oracle 服务报告的章节体系。
- **Oracle 错误码全集**（`msg/`）：ora-10g/11g/12c + tns + crs 五个版本的官方消息库，39.3 万行，附 `oerr` 的 Perl 重实现。Oracle ACE 的家底。
- **装机前检查**（`sc_check/`）：Linux/AIX/HP-UX 的系统参数、Oracle 参数、空间、SELinux、时区逐项核查——传统 DBA 服务公司的标准作业程序，代码化了。

### 3.4 LLM 层：不在这里

全仓 grep `deepseek|openai|llm|prompt`，命中的只有 Python3 安装脚本（装依赖时提及）。**论文的核心增量——相关性异常模型的 DNF 检测、两阶段图探索、推理 LLM 五段式报告——在这个版本里没有实现代码**，只在 `Appendix_Diagnosis_Report.pdf`（6 页报告样例）和 README 里作为系统能力出现。合理推断：LLM 诊断跑在闭源服务端（dbaiops.com 社区版也只放采集工具链），开源边界划在"底座"。

这不一定是坏事——恰恰是商业上清醒的开源策略：采集适配和报告模板没人愿意白写（7.6 万行是真成本），而 LLM 链路给了也跑不起来（需要图谱数据和 prompt 资产）。但读论文时必须知道：**你看到的评测数字，跑在你看不到的那半边系统上**。

## 四、两个被 master 分支弄丢的评测文件

这个 commit 里有两个文件值得单拎出来，它们回答了上篇精读里"评测细节偏薄"的批评——细节其实开过源，只是后来丢了。

**HEval_criteria.md**（论文引用、最终 404 的评分标准原文）：
- 明确"语义覆盖 ground truth 根因"而非字符串匹配；
- 三档打分制：High / Medium / Low / Unsatisfied，每档给分数区间（如证据真实性 High 31-40、Low 1-10）；
- **门槛逻辑**：根因召回不达标直接总分 0，理论一致性和证据真实性只在召回达标后适用——先问"答对没有"，再问"推理对不对、数据真不真"。

**LLM_output_similarity.md**（论文未展开的防御性验证）：
- 用 difflib / Levenshtein / Jaccard / TF-IDF 四个相似度指标，比对 LLM 输出与输入图谱内容；
- PG 场景相似度 0.10-0.37——证明报告是推理生成而非从图谱复制粘贴， preemptively 防"RAG 复读机"质疑。这是 KG+LLM 论文该做但极少有人做的自证。

## 五、批判性解读

1. **"开源系统"的边界感**：论文实验系统 ≠ 开源代码。底座全给你（采集、元数据、图谱构建、报告模板），大脑不给你（LLM 诊断闭环）。引用它做复现研究时，缺口在 prompt 设计和图谱数据本身——而图谱数据因 AES 加密也不随仓库分发。
2. **工程债清晰可见**：`update_knowl.py` 硬编码内网 IP（192.168.32.98）和 Neo4j 明文密码；CQL 用 Python 字符串 `%` 拼接（注入风险换便利）；AES 密钥写死在源码里；`bin/hola` 是残留的测试 shim。典型"交付优先"的国内 to-B 软件质感——功能全、边界糙。
3. **知识生产方式与论文叙事有缝**：工程底座=DBA 维护 Excel；论文叙事=LLM 从 MOS 半自动抽取。两处都对，但中间那步（LLM 抽取器）恰好是没开源也没给评测的部分。
4. **664 张表的平台本体才是隐形资产**：多数 KG+LLM 论文从监控数据直接跳诊断，DBAIOps 底座里那套 AWR 明细、基线画像、采集任务调度、健康检查扣分模型，是诊断质量的地基——论文一句话带过的"数据收集使用 DBAIOps 社区版工具"，指的就是这层 22 万行 DDL。
5. **版本治理教训**：发表后用完全不同形态的 artifact 覆盖 master 分支，导致论文引用断链（HEval 404）、评测数据不可复现。如果用户想引用这个版本，**必须钉死 commit 号 cfe9d30**——这正是用户做的事情，值得写进方法论。

## 六、对建设者的迁移清单

- **采集合乎规范先行**：60 个脚本覆盖 25+ 种库，cib/metric/health 三类分离——没有这层，图谱和 LLM 都是空中楼阁。
- **知识资产化的正确姿势**：Excel（人维护）→ 同步脚本 → 图库，带加密字段保护核心知识；新库接入=新增 Excel 行+一列标签，这个流程比"端到端 LLM 抽取"便宜一个数量级。
- **参数模板矩阵直接可用**：按规格×形态组织 GUC 模板的设计，对任何做多实例数据库平台的人都是现成范式。
- **评测三件套抄走**：HEval 门槛逻辑、相似度自证、考题不进题库（见上篇）。
- **红线**：别把内网 IP、密码、密钥 commit 进仓库——这个仓全犯了。

## 参考资料

1. [DBAIOps 仓库（OpenDataBox），commit cfe9d30](https://github.com/OpenDataBox/DBAIOps/tree/cfe9d30f5e6c2b3a970d4c8d888b3f5a0476041f)——本文全部代码证据来源
2. [DBAIOps 论文原文（PVLDB 19(6): 1319-1331，2026）](https://www.vldb.org/pvldb/vol19/p1319-zhou.pdf)——系统对应的学术论文
3. [DBAIOps 论文精读（本博客，2026-10-09）](./2026-10-09-dbaiops-paper.md)——姊妹篇，评测方法详解
4. [HEval_criteria.md（cfe9d30 版本内文件）](https://github.com/OpenDataBox/DBAIOps/blob/cfe9d30f5e6c2b3a970d4c8d888b3f5a0476041f/HEval_criteria.md)——人工评估评分标准原文；master 分支已丢失
5. [LLM_output_similarity.md（cfe9d30 版本内文件）](https://github.com/OpenDataBox/DBAIOps/blob/cfe9d30f5e6c2b3a970d4c8d888b3f5a0476041f/LLM_output_similarity.md)——LLM 输出相似度自证数据
6. [Appendix_Diagnosis_Report.pdf（cfe9d30 版本内文件）](https://github.com/OpenDataBox/DBAIOps/blob/cfe9d30f5e6c2b3a970d4c8d888b3f5a0476041f/Appendix_Diagnosis_Report.pdf)——6 页诊断报告 case study（投稿版占位抬头）
7. [佰晟智算完成千万元级天使轮融资（36氪，2025-12-10）](https://m.36kr.com/p/3589507644047625)——团队与"20 万条结构化运维知识"口径（融资稿，未经审计）
8. [DBAIOPS 社区官网](https://www.dbaiops.com)——社区版工具与图谱平台入口
