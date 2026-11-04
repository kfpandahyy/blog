# Supabase 收购 Turso 全解析：Postgres 巨头为什么买下一个 SQLite 公司

2026 年 10 月 2 日，Supabase 在同一天扔出两颗炸弹：完成 1.5 亿美元新融资（GIC 领投，Alphabet 旗下 CapitalG、IronArc、SquarePeg 跟投），并官宣收购数据库创业公司 Turso。交易价格未披露，整合时间表未公布，但官方给这笔收购定的方向非常明确——为 AI Agent 时代准备数据库基础设施。

这篇文章只做一件事：把"为什么"讲透。数据与事实均标注来源与截至日期（2026-11-05），厂商自述口径单独说明。

## 一、先厘清事实：买的不是 SQLite，是 Turso

先纠正一个流传中的表述误差：Supabase 收购的不是 SQLite。SQLite 本体以公有领域形式发布，没有任何公司能"买下"它。被收购的 Turso 手里有三样资产：

1. **libSQL**：SQLite 的 Rust 语言重写版，开源。因 SQLite 许可证特殊（公有领域但贡献条款严苛），Turso 一早选择了 fork 路线，并在重写中加入了原生 SQLite 没有的能力——最核心的是 **MVCC（多版本并发控制）**，打破了 SQLite 单写者的历史限制，使其可以面向服务器负载。
2. **云平台**：一套 diskless、WAL 直接落在 S3 对象存储上的架构，官方与客户口径显示单台服务器可以管理数以百万计的数据库，支持公有云与 BYOC（客户自带云）两种部署。
3. **团队**：创始人 Glauber Costa、Pekka Enberg 及工程团队整体加入 Supabase。Glauber Costa 出任 Supabase **Head of Agentic Services**——注意这个头衔，后文会用到。

交易条款本身：价格未披露，无交割时间，无 SLA 承诺。官方口径是 Turso 品牌保留、平台继续运营、代码保持开源，现有用户"nothing changes"，Turso 用户将来应用长大后有一条"毕业"到 Supabase 标准 Postgres 的路径。已披露的 Turso 客户包括 Superhuman、Sauna.ai、CTO.new、Mastra 等 Agent 基础设施类公司。

资本背景值得看一眼节奏：2025 年 10 月 E 轮估值约 50 亿美元 → 2026 年 6 月 F 轮 5 亿美元、投后 105 亿美元（GIC 领投，Accel、YC、Craft、Felicis、Peak XV、Coatue、Stripe、Salesforce Ventures 跟投）→ 2026 年 10 月 2 日 1.5 亿美元延展轮 + Turso 收购。一年之内估值翻倍，三轮操作指向同一个增长故事。

## 二、动因一：建库的主体，正在从人变成 Agent

这笔收购的第一个为什么，藏在 Supabase 自己的运营数据里（以下均为官方自述口径，未经独立审计）：

- 平台每月新增 **100 万用户、400 万+ 数据库**（2026-10-02 新闻稿）；
- 其中 **70% 的新数据库由 Agent 或 AI 工具创建**；
- 数据库总量同比增长 **600%**（2026-06 F 轮时披露）；
- 2026 年以来，**Claude Code 是平台新增使用的最大单一来源**。

最后一个数据点尤其说明问题：Supabase 增长最快的客户 segment，不是某类开发者，而是一个 AI 编程 Agent。Bolt、Lovable、Cursor、Figma 这类 AI 编程工具把 Supabase 作为默认后端预置，意味着大量应用在"创始人还没做任何基础设施决策"之前就已经长在了 Supabase 上——建库这个动作，正在从人的主动行为变成 Agent 的自动行为。

Supabase 对终局的判断是：现在的曲线指向**每月数十亿个数据库**，而其中的绝大多数很小——是一个 Agent 会话、一个分支预览、一个 vibe coding 出来的原型，用一阵子就丢掉。Copplestone 在大会访谈里把危险时刻说得非常具体：一个应用冲上 Hacker News 首页时，流量会瞬间超出原架构的承载能力——平台必须在此之前就给出升级通道。

## 三、动因二：Postgres 的经济模型，撑不起"一 Agent 一库"

第二个为什么是成本结构问题，也是最本质的一个。

Postgres 是"每库一实例（或一进程组）"的架构：一个库意味着独立的连接管理、内存预留、存储挂载、备份周期。这套模型服务"几千到几万个认真运行的应用"没问题，但当需求变成"每次 Agent 会话都要即时开一个隔离的库、用完即弃"时，单库成本就变成了死结——你不能为一次几分钟的会话开一个常驻 Postgres。

SQLite 在这个问题上是个异类：**它是一个文件格式，不是一个服务**。一个库就是一个文件，没有常驻进程，开销趋近于零，可以毫秒级创建、挂起、销毁。这正是"按需按 Agent 建库"唯一被大规模验证过的粒度。Supabase 官方博客的判断是：这种规模与模式下，需要一次数据库基础设施的演进。

Turso 把 SQLite 的文件格式经济学和云服务揉在了一起，补上了三块关键拼图：

- **MVCC 多写者**：解决原生 SQLite 的单写者瓶颈，让文件格式能扛服务器负载；
- **Diskless + WAL-on-S3**：数据库本体无状态，WAL 日志落在对象存储上，S3 的 11 个 9 持久性充当地基，换来极致的密度——单服务器管理百万级库；
- **BYOC 与公有云双形态**：Agent 厂商既可以用 Turso Cloud，也可以把这套架构开进自己的云账号里。

成本上的锚点：按官方对外叙述，这套架构让每个库的供给成本降到传统方式的"零头"（a fraction of conventional costs）；第三方报道显示 Turso 现行有**个位数美元每月、不限数据库数量**的档位（beri.net 报道为 4.99 美元/月不限库数）。无论精确数字如何，"按库计费"与"按实例计费"是两个数量级的生意。

## 四、动因三：一篇博客点燃的收购——买的是路线图

第三个为什么，外人很少注意到：这笔收购的直接导火索是一篇博客。

在 Supabase Select 2026 大会前夕的 Latent Space 播客里，两位联合创始人明确说：真正触发收购决定的，是 Turso 团队发布的那篇描述**如何重写架构、从而让 SQLite 文件格式打开兼容 Postgres 事务语义路径**的博客。换句话说，Supabase 买的不只是当下能跑的产品，而是"SQLite 文件格式与 Postgres 世界打通"这条技术路线的独家船票。

这个细节解释了为什么是"收购 Turso"而不是"自己做 libSQL"：SQLite 的许可证问题注定了必须 fork，而 fork 之后的多年工程——Rust 重写、MVCC、diskless 架构、与对象存储新特性的磨合——不是招几个人两年能追上的。买，是用一笔未披露的钱换两三年的时间差。

## 五、放在棋局里看：四连发拼出的是一条升级通道

单看这笔收购容易低估它。把它放回 2026 年 10 月 Supabase 的连续动作里，图就完整了。同期发布的还有三样东西：

- **Supabase Compute**：无服务器、长时运行的计算实例，与数据库同私有网，跑 Python/Rust 乃至 Agent，替掉原本只有几秒执行额度的 Edge Functions——这是 Agent"干活"的位置；
- **OrioleDB**：替换 Postgres 默认存储引擎，性能提升 1.5–2 倍，完全兼容现有生态；更狠的是商业操作——Supabase 收购了这项技术、**买断了相关专利**，然后以 Apache 2.0 全量开源；
- **Multigres**：由 Vitess 联创 Sugu Sougoumarane 主导的 Postgres 分片与 operator 体系，设计伙伴已含 Postman、Perplexity；团队对生态内主流 Postgres operator 的测试结论是"特定条件下每一个都可能丢数据"，Multigres 的目标是 1000 次连续故障切换不丢数据、且分片上线后查询零改动——这是向上扩容（scale out）的答案。

加上 Turso，四件事指向同一个命题：补上 Postgres 在云上最缺的几块拼图，而且**向上、向下两个方向都要能走**。

向下这一半就是 Turso 的角色：Supabase 推出新的轻量实例类型 **Supabase Spark**——任何想法、任何原型，先零成本拿到一个数据库，与现有全部库兼容；长大了无缝升级成完整 Postgres；再上规模接入 Multigres。Copplestone 那句"一旦你上了 Hacker News 首页，你得知道你还能走多远"，翻译过来就是：**升级通道必须零停机，而 Turso 是这条通道的起点**。

没有 Turso，Supabase 只能接住"长大了的应用"，接不住"刚出生的应用"——而在 Agent 批量造应用的时代，出生点恰恰是最关键的入口。

## 六、竞争与防守：这是一场入口之争

第四个为什么是战略防守。Agent 时代的数据库入口正在被重新定义，而且对手已经动了：

- **Databricks 收购 Neon**（2025 年 5 月，约 10 亿美元）：押注"数据库由代码创建"——Databricks CEO 公开说 Neon 平台上五分之四的数据库是代码开的，不是人开的；
- **Cloudflare** 持续推进 Workers/Containers 的预览与沙箱能力，容器启动时间从约 4 秒降到 648 毫秒，方向同样是"海量短命执行环境"；
- **Deno Sandbox** 等产品在 Agent 隔离运行环境上持续加码。

如果"每个 Agent 一个轻量 SQLite"成为 Agent 基础设施的默认架构，Postgres 中心的平台就会丢掉整个漏斗的顶端。Supabase 买下 Turso，等于把这个入口也买了下来：Spark 在应用出生时接住它（成本趋零）， graduation path 保证它长大后的每一笔收入仍落在 Supabase。这是典型的"在最便宜的地方建立关系，在最贵的地方收获收入"。

对 Turso 一侧，这同样不是失败性退出，而是换了一个能把其架构送进更大流量入口的买家——前提是 Supabase 兑现"品牌保留、继续开源"的承诺。

## 七、边界与风险：必须摆在桌面上的五件事

1. **交易条款全黑箱**：价格、股权/现金结构、交割时间均未披露，外界无法判断交易规模，也无法判断整合深度会到哪一层。
2. **增长数据是厂商自述**："70% 由 Agent 创建""月增 400 万库"来自 Supabase 官方，未经第三方审计，但该数据同时是估值逻辑的核心支柱——这是本次叙事中可信度最需要打折的一环。
3. **"现有用户一切照旧"是单方承诺**：不是合同条款。尤其值得注意的是，按库计费的成本设计现在归 Supabase 所有，而双方都没有书面承诺现行价格不变（beri.net 的质疑点）。Turso 用户的理性做法是把本次变更记入供应商风险清单。
4. **双引擎分工的天花板**：官方明确，超大规模事务型负载 Turso 接不住，仍靠 Multigres 路线；SQLite 适合"海量、小型、按需"，长期化、强审计、跨会话一致性的数据仍应留在 Postgres——这不是谁强谁弱，而是架构边界。
5. **社区变量**：libSQL 社区对"被一家 Postgres 公司收购"的反应、以及开源承诺的兑现节奏，将决定这套架构能否继续保持中立生态位。

## 八、判断

这笔收购的本质，是用一笔未披露的钱，买断"Agent 时代数据库入口"最有竞争力的候选方案。

拆开看有三层收益：技术上买到已经完工的架构（文件格式经济学 + S3 地基 + MVCC），产品上买到漏斗顶端（Spark 承接 99% 长不大的微型库），战略上买到时间（自研同等能力需数年，而 Cloudflare、Deno、Databricks 都不会等）。

风险同样清楚：整合无公开时间表，关键增长数据依赖厂商自述，社区承诺待兑现。这场仗的终局不取决于公告，取决于 12 个月内两件事——Spark 是否如期落地且价格真正亲民，以及"SQLite 兼容 Postgres 事务语义"的技术路线能否兑现。前者验证商业闭环，后者验证技术赌注。

对行业的启示比交易本身更大：当建库主体从人变成 Agent，数据库竞争的计量单位就从"实例"变成了"库"。在这场新计量体系里，密度与供给速度取代功能丰富度成为第一竞争力——这正是 Postgres 巨头要向一个 SQLite 公司买答案的原因。

---

## 参考来源

1. Supabase 官方博客：Supabase is acquiring Turso，2026-10-02。https://supabase.com/blog/supabase-is-acquiring-turso
2. PR Newswire：Supabase Announces $150M in New Funding and Turso Acquisition，2026-10-02。https://www.prnewswire.com/news-releases/supabase-announces-150m-in-new-funding-and-turso-acquisition-302896752.html
3. SiliconANGLE：Database startup Supabase raises $150M, acquires Turso，2026-10-02。https://siliconangle.com/2026/10/02/database-startup-supabase-raises-150m-acquires-turso/
4. 澎湃新闻：Supabase 四连发：换掉 Postgres 存储引擎，收购 Turso 押注 SQLite，2026-10-18。https://www.thepaper.cn/newsDetail_forward_34266114
5. 至顶网：Supabase 用六年时间憋出的三张底牌：性能、可靠性与十亿级数据库，2026-10-27。https://www.zhiding.cn/aidc/2026/1027/3203667.shtml
6. Apifox 专栏：Supabase 收购 Turso：为每个 Agent 现开一个 SQLite 数据库，2026-10-08。https://apifox.com/apiskills/supabase-acquires-turso-sqlite-agent-databases/
7. daily.dev：Supabase Select 2026: Turso acquisition, Compute, Multigres and OrioleDB，2026-10-09。https://daily.dev/posts/supabase-select-2026-turso-acquisition-compute-multigres-and-orioledb-kvn8bdkx4
8. beri.net：Supabase Buys Turso and Its $4.99 Unlimited-Database Plan，2026-10-03。https://www.beri.net/article/supabase-turso-acquisition-database-per-agent-sqlite-postgres-pricing-continuity
9. Koala OSS 周刊：Supabase 收购 Turso，2026-10。https://koala-oss.app/news/1771/
10. ecorpit：Neon vs Supabase in 2026（含 Databricks 收购 Neon 交易背景），2026-08-02。https://ecorpit.com/neon-vs-supabase-serverless-postgres-saas-backend-2026/

*注：文中"70% 新库由 Agent 创建""月增 400 万+数据库""600% 同比增长"等为 Supabase 官方自述数据；4.99 美元/月不限库数档位来自第三方报道（beri.net），均未经独立审计。融资与估值信息以 PR Newswire 通稿及公开报道为准。数据截至 2026-11-05。*
