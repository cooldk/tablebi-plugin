---
name: tablebi
min_cli: 0.4.1
description: >-
  The data backend for your Claude Code. PIPE marketing sources in (Google Search Console,
  GA4, Google Ads, Meta Ads, CSV), ASK in two altitudes — unified cross-channel metrics
  (ROAS/CPA/CTR/spend/clicks) or the raw per-platform tables in SQL — and PIN the answer as a
  live, self-refreshing dashboard URL. Drives the `tablebi` CLI (no MCP needed). Use when the
  user says "connect / import Search Console, GA4, Google Ads, Meta or a CSV", "how is this
  campaign doing", "which campaigns waste budget", "ROAS by channel", "build / publish a
  dashboard", "give me a live link", "is the data fresh", "catch me up on this account",
  "watch this site every week", "is this page indexed / any sitemap errors". 中文触发:
  连 / 导入数据源、这个 campaign 怎么样、哪个在浪费预算、按渠道看 ROAS、建 / 发布看板、
  给我个活链接、数据新不新、把这个账户的情况同步给我、每周盯着这个站、这个页面收录了没有。
---

# TableBI —— Claude Code 的数据后端(Pipe → Ask → Pin)

tablebi 是名词(数据和看板的家),你是动词(驱动它的手)。脑永远是你;tablebi 不托管 LLM。
**Pipe** 接入 + 口径归一 → **Ask** 问数(两个高度)→ **Pin** 固化成公开只读、自刷新的看板 URL。

## 开场先 rehydrate

每个会话先跑一次 `tablebi context --json`(口径包 `grammar.packs` + 已连源 + 新鲜度 + 工作区定义 `definitions` + 看板 + 账号 + 命令签名)。数据为空就先 `tablebi connect …`。

命令若在 stderr 打印「正在准备你的数据环境…」是**正常等待**,CLI 会自己重试;只有最终返回 `status:"unavailable"`(退出码 3)才是真没好,照它的 `remediation` 稍后重跑同一条命令。

**多站 / 组合解读 —— 别把 N 个站说成 1 个:** 同一平台常连了多个站 / 账户(GSC 一个授权下 N 个验证域名)。`context` 的 `data.sites` 按 account 逐行给 `site`/`rows`/`dataThrough`/`ageDays`/`stale`,GSC 站另有 `clicks28d`(近 28 天站点总点击)与 `anonPct28d`(匿名占比 %:按搜索词拆开时看不到的那部分);GSC 站按 `clicks28d` 排在前,其余按行数,**只列前 20 个**;全部看 `tablebi sources` 或 `context --full`。

- 报「连了 N 个站」用 `data.siteCountByPlatform`(GSC 站与 GA4 属性分开数),别拿混在一起的 `siteCount` 当站数;**别**把平台级总行数 `data.rows` 当成某单站的行数,也**别**只报 `sample` 里碰巧出现的那个 account——那是采样,不是「唯一的站」。
- 平台级 `trust.freshness` 取各站 max,**会被一个跑得快的小站粉刷成新鲜**。数据真到哪天看 `coverageThrough`(所有主要站点都覆盖到的那天)与 `laggingSites`;站级陈旧计数看 `data.staleSites`。旧站单独 `sync`(GSC 用 `--site`)。

## 首次同步的 1–2 分钟:别让用户干等

首次连源拉数较慢。**别只甩一句"正在同步"就沉默**——用这段时间就着即将到来的数据先聊:确认连了哪个账户/站点、马上能看什么、你打算先帮他看哪条。说人话,**绝不暴露实现词**(没有 pod/warming/binding 这类词)。

- ✓「正在同步 plairmoi.com 最近的数据,大概一两分钟。等它好,我先帮你扫一眼各渠道 ROAS 有没有在烧钱的 —— 你最担心哪块?」
- ✗ 发一句"同步中…"然后干等

## 刚连完 / 同步完:就着真数据给钩子(激活的关键一跳)

**别甩"你可以问 ROAS / 看趋势"这种泛泛能力清单**——那是最弱的教育。先用一两个 `query` 探一眼真实数字,再给 **2–3 个就着具体数、带钩子的下一步**。这一步直接决定新用户会不会走下去。

**先探一眼——按已连的源选探法,别一律打 ROAS**(ROAS/CPA 是**投放口径**,只对连了 Google Ads / Meta 的用户有意义):

- 有投放源 → `tablebi query --kind table --view metrics --metrics cost,roas,conversions --by platform --json`(再按 campaign 看异常花费)。
- 只有 GSC → **别提 ROAS/花费**:`tablebi query --kind compare --view search_console --metrics clicks,impressions --by page --window last-7d --json`(页面涨跌;`--by query` 看词)。
- 只有 GA4 → `tablebi query --kind compare --view ga4 --metrics sessions,conversions --by channel --window last-7d --json`。

**再给钩子**(具体带数 > 泛泛能力):

- ✓「你 Brand Search 这周 ROAS 只有 0.9×、还烧了 $1.2k —— 要我拉出来看是不是在浪费预算?」
- ✓「/best-summer-dresses 点击涨了 40%,要不要 pin 一张盯它的看板?」
- ✗「你可以问各渠道 ROAS」(没数、没钩子;对没投放的用户连提都别提)

用户点头就**顺势直接做**。诚实:只说数据真支持的(时间吻合的相关,别吹因果);拿不准就说"看着像,要不要深挖确认"。

## 数据薄 / 几乎为空:冷启动陪跑(别硬找问题)

**判定看 `context` 的 `data.thin`**(true = 无投放源且近 90 天活跃不足几百;服务端算好的,别自己目测)。数据薄时上面的钩子模式**无钩可给,别硬挤**——硬找的"发现"是垃圾洞察,一次就毁信任。换这套:

1. **诚实开场,换价值主张**:「你的站还年轻,现在数据还撑不起'分析'——但正因为刚起步,从第一天记录在案才最值:三个月后你能看到完整成长曲线,每个动作对应哪段涨跌都有据可查。」
2. **把能连的都连上**(GA4 / CSV 历史台账),基线越全将来越有据。
3. **以默认看板为起点**:系统已自动生成一张(console 首页那张)。`tablebi dashboard spec weekly` 读定义、改完 `pin` 成自己的;别再 pin 一张一样的(它每次同步会被覆盖)。
4. **提议每周检查**(见「定时盯盘」):薄数据用户的留存靠每周一个小确幸,不靠深度。
5. 每周钩子说**成长**不说问题:✓「这周有 3 个词进了前 50,/pricing 第一次有自然点击」 ✗「你的 ROAS…」(没投放,永远别提)。

数据长起来(有投放、或 GSC 月点击上千)就自然切回钩子模式。

## Ask:先 `query`(语法),答不了再 `ask`(SQL)

`tablebi query` 只说**要什么**(kind · view · metrics · by · window · compare · filter),编译器负责挑表、锚日期、在同一聚合层上算比值、排除品牌词。返回行 + **生成的 SQL** + 时间窗 + trust —— 问的数和钉进看板的是同一个编译器算的。

```bash
tablebi query --kind kpi --view search_console --metrics clicks,impressions,ctr,avg_position --compare previous_period --json
tablebi query --kind breakdown --view search_console --metrics clicks,ctr --by query --filter non_brand --limit 20 --json
tablebi query --kind trend --view ga4 --metrics sessions --split-by ai_assistant --filter ai_only --window last-90d --grain week --json
tablebi query --kind compare --view metrics --metrics cost,roas --by platform --json
```

- kind:`kpi` 一行总数(≤ 4 个指标,加 compare 出环比)· `trend` 随时间 · `breakdown` 哪些最大 · `table` 明细 · `compare` 本期 vs 上期。
- 每个包有哪些维度 / 指标、这个工作区实际有哪些表:`context` 的 `grammar.packs`(只列已连的)。GSC 的表路由编译器做:总数走站点总数表、按词走明细 —— 返回的 `caveats`(如「明细不含匿名查询」)转述时带上。
- window 默认 `last-28d`,**锚在数据覆盖到的那天**(不是今天);按周 / 按月的趋势只画完整的桶。filter:`维度=值`(`*` 通配、逗号 = 或)、`<维度>_not=` 排除、`site_group=` / `exclude_sites` / `non_brand` / `brand`(GSC)、`organic` / `ai_only`(GA4)。
- 报错会列出可用的值,照改即可。语法表达不了的(自定义算法、跨表 JOIN)再用 `ask` 写 SQL。

### SQL(`ask`)的两个高度

- **口径**:视图 `metrics`(跨渠道统一,营收列名 `revenue`)+ 宏 `roas` / `ctr` / `cpc` / `cpm` / `cpa` / `cvr` / `aov` / `avg_position` / `platform_label` / `ai_assistant`。比值是**聚合后**算的,用宏,别手搓。
- **原生**:`<platform>_raw`(单平台全字段;广告两家的原生表只到 campaign 级,Google Ads 花费是 `cost_micros`)、统一裸表 `facts`(营收列名 `conversion_value`)。`context` 的 `sql.views` 是当前可查的视图,`tablebi context --sample` 看真实列名。
- **GSC 四张表各管一类数**(别拿明细加总当总数):总数 / 趋势 → `search_console_totals_raw`(**加 `search_type = 'web'`**,含匿名查询 = 后台总览);页面 → `search_console_pages_raw`;国家 / 设备 → `search_console_geo_raw`(`country` 是 ISO-3 小写如 `usa`);搜索词 → `search_console_raw`(**不含匿名查询**,各站差多少看 `data.sites[].anonPct28d`)。平均排名一律 `avg_position(SUM(position*impressions), SUM(impressions))`。
- **GA4**:渠道 / 国家 → `ga4_raw`;来源 × 媒介 → `ga4_sources_raw`(AI 助手引荐加 `WHERE ai_assistant(source) IS NOT NULL`);AI 引来的落地页 → `ga4_ai_landing_raw`。`conversions` 都是 GA4 的 key events。

`--json` 给你解析。**只读**:一条 SELECT / WITH,引擎锁死(读不了别的文件 / 网络,超时被杀)。

**追问顺序按 `context` 的 `drill.next` 走**(定了站 → 问词 / 页;定了词 → 问页;定了页 → 问词)。看板上的「点即筛」读的是同一张表:用户点一行、那一维被筛定,剩下的正是 `drill.next` 里那几个 —— 按它追问,你和用户点出来的是同一条路。

## 定义:把每次都要重说的常识沉淀下来

`tablebi define` 无参数 = 列出生效值(`defaulted` = 还在用默认值的键)。用户说出这类常识(「tabl 开头的都是品牌词」「这两个属性是同一个站」「lokuma 那个站别算」)就**当场 define**,别在每条 SQL 里手写 `NOT IN` / `VALUES` 映射 —— 发布时 lint 会拦。

- 品牌词 `tablebi define brand_terms "acme*, acme corp"` → filter `non_brand` / `brand` 可用,机会词不再算品牌词;
- 站点组 `tablebi define site_group core "a.com, b.com"` → filter `site_group=core`;总览别算的站 `exclude_sites "old.com"`;
- GA4 属性显示名 `property_alias "543715615 = tabledi.com"`(两个编号同名 = 合并成一行);站 ↔ 属性 `site_property tabledi.com "543715615"`(每站看板归属错了时);
- 阈值 `striking_distance "4-15, 5"`(机会词)、`movers "min_clicks=5"`(涨跌榜);转化口径 `ga4_conversion_events "purchase, sign_up"` / `meta_conversion_actions "purchase, lead"`(下次同步起生效)。
- 看板页面语言 `language`:**新空间默认 `en`**(公开看板的标题、表头、角标、页脚出英文);用户用中文跟你说话、看板给中文读者看时 `tablebi define language zh`。`query` 的输出与快照里的列名不随它变(照旧中文,给你读);`dashboard template` 出的模板跟着它带 `defaults.lang`。

系统默认看板(工作区总览 + 每站)按这些定义出段;改了会在后台重建。删一个用 `--unset <键>`。

## GSC 直连:同步表答不了的才用

`tablebi gsc query|inspect|sitemaps` 由服务端代你调 Google(凭证不出服务端),只能查**本 workspace 已连接**的站。

1. **先 `ask`**:总数 / 趋势 → `metrics` 或 `search_console_totals_raw`(`search_type = 'web'`);页面 → `search_console_pages_raw`;搜索词 → `search_console_raw`;国家 / 设备 → `search_console_geo_raw`。
2. **同步表答不了才用 `tablebi gsc query`**:词×国家/设备、图片或 Discover 明细、搜索外观、小时级、同步窗口之外的明细。
3. **别用直连复制同步表里已有的数**:浪费 Google 配额,还可能拖慢当天的同步。
4. **`tablebi gsc inspect` 只查少量关键 URL**(每次 ≤ 20 个,每站每天 200 个),别批量扫站。

```bash
tablebi gsc query --site chatdiagram.com --filter "country = usa" --filter "device = MOBILE"   # 美国移动端的搜索词
tablebi gsc query --site nichelogo.app --type image --dims page                              # 图片搜索带来的页面
tablebi gsc query --site chatdiagram.com --dims query,page --filter "page contains /tool/"            # 某个目录下的词
tablebi gsc query --site chatdiagram.com --dims hour --days 1                                # 今天按小时
tablebi gsc inspect --site chatdiagram.com https://chatdiagram.com/pricing                   # 收录没有、canonical 对不对
tablebi gsc sitemaps --site chatdiagram.com                                                  # sitemap 抓了没有、有没有报错
```

- 默认 `--dims query`、`--type web`、近 28 天(到昨天;按 hour 分组时含今天)、`--limit 1000`(最多 25000,更多用 `--start-row` 翻页)。多个 `--filter` 之间是 AND;操作 `= != contains !contains regex !regex`,正则是 RE2。`country` 用 ISO-3 小写(`usa`),`device` 是 `DESKTOP` / `MOBILE` / `TABLET`。
- **两种写错了 Google 也不报错、只回空**:页面过滤里写死主机(页面常在 www 或子域上,按目录过滤用 `page contains /目录/`);正则语法错。结果为空时先看 `notes`。
- 返回的 `notes` 是口径提醒(按词分组不含匿名查询、哪天起是初步值、结果有没有截断),转述数字时带上;`truncated: true` 说明还有更多行。`request` 回显了实际查的窗口。
- 限制:同时按 query 和 page 分组或过滤,一次最多 93 天;最早查到 16 个月前;每个 workspace 每分钟 20 次;相同请求命中缓存(`cached: true`)。
- 报错:400 带着 Google 的原话(多半是维度组合不合法),照它改参数;403 → 请用户重新 `tablebi connect gsc`;429 → 照 `remediation` 等(Google 配额要等 15 分钟,每分钟限速只需几秒),别重复同一查询。

## Pin:固化成活看板

把一组查询钉成公开只读 URL(每次打开按**当前数据**重算,不是死截图)。

**主路径:v2 spec 文件 + `dashboard publish --file`。** widget 用和 `query` 同一套语法;图型由 kind 定(kpi → 记分卡、trend → 折线、breakdown → 横条,可用 `"chart": "bar" | "pie"` 改),时间窗进每格的角标,标题可省(由语法生成,≤ 40 字)。

```bash
tablebi dashboard template seo-overview > /tmp/board.json   # 按已连的源出模板;无参数列出全部
# 改 /tmp/board.json:删段、加段、改 window / filter / limit
tablebi dashboard publish --file /tmp/board.json --dry-run  # 每个 widget 编译 + 真跑 + lint
tablebi dashboard publish --file /tmp/board.json            # 建板并发布;改已有的:dashboard publish <token> --file …
```

形状:`{ "version": 2, "title": "…", "defaults": { "view": "search_console", "window": "last-28d" }, "widgets": [ { "kind": "kpi", "metrics": ["clicks", "ctr"], "compare": "previous_period" }, { "kind": "breakdown", "metrics": ["clicks"], "by": "page", "limit": 15 } ] }`。
每板 ≤ 12 个 widget(硬上限 16);同构的多张(每个站 / 属性一张)用 `split_by` 合成一张。建看板默认**先来一排 kpi + 一两张趋势**,别交一堆纯表格。

**SQL 是逃生舱**:`{ "kind": "sql", "sql": "…", "title": "…", "chart": "line" }`,页面标「自定义查询」,v2 看板里要过 lint(不许内嵌 VALUES 映射表、`NOT IN` 超过 3 项、`current_date` / `now()`、没有 LIMIT 的 GROUP BY)。旧格式 `{ title, widgets: [{ title, sql, chart }] }` 与 `tablebi pin --file` 照常可用(v1 永远照读,lint 只警告);图型权威在 `context` 的 `chartKinds`。

**Pin/publish 完不要只甩 token —— 把人引回 console。** 主动告诉用户:看板是**活的**(自刷新),公开只读链接可直接发给合伙人/客户;它也已**陈列在 console.tablebi.com → 看板**里,去那看全貌、管理、分享。对话负责创作,console 负责陈列与分享,做完就把人送回载体。

## 定时盯盘:pin 完主动提议「我每周替你看一眼」

没人天生记得回来看数——看板 pin 完,**主动问一句**:「要不要我每周一自动检查这张板,有大变动叫你?」用户点头就用你宿主的定时能力建周任务(Claude Code 用 scheduled tasks / cron):

1. 跑 `tablebi context --json`(顺带触发保鲜)+ 把看板核心指标和上周对比;多站 GSC 组合先 `tablebi scoreboard --json`(谁涨谁跌一眼出,省 N 条 ask)。
2. **有异动才开口**(某指标周环比超 ±20%、某源 stale):一句人话 + 数 + 下一步建议。
3. **怎么"叫人"**:定时任务里跑、用户不在对话 → 要有推送通道。**装了 clize** 就发一封简短邮件:一行异动 + 数 + **一句可粘回对话的话** + 看板链接(`clize email send`;发真人是硬闸,**先把草稿给用户看、他说发再带 `--confirm`**)。没 clize 就记下,下次对话主动说。用户就在对话里 → 直接说,别绕邮件。
4. **没异动就别刷存在感**(不发"一切正常"的水报告);数据薄的站例外——每周报一个成长小确幸。

这是 tablebi 形态独有的:别家托管 AI 只能发个死通知,你是能接着干活的 agent——发现异动当场下钻、当场改看板,邮件只是把人拉回对话的钩。默认频率**周**,用户嫌烦一句话就撤。

> 备注:这条是**用户自己 agent 层**的召回(BYO,零后端);产品侧另有常开后端 digest 邮件兜底,默认关、需另配,不在本 skill 职责内。

## 断开数据源 / 删除工作区(不可撤销)

用户说「断开 / 移除这个站 / 这个账户」「删掉我的数据」「删掉这个工作区」时:

1. 先不带 `--yes` 跑 `tablebi disconnect <provider> --site|--account|--platform <目标> --json`:它**什么都不删**,只回 `{status:"confirm_required", effect, next}`(退出码 2)。把 `effect` 原样讲给用户,**得到明确同意**再跑 `next` 里那条(带 `--yes`)。
2. 断开 = 停止同步 + 删掉这个源已同步的全部数据;它是该类型最后一个源时授权也一并删(平台那边的授权撤销,把返回的 `note` 告诉用户)。CSV 用导入时的 `--platform` 标签。
3. 删整个工作区:`tablebi workspace delete <名> --confirm <名>`——数据、看板(公开链接随之失效)、授权全部删除。只在用户**亲口**说要删整个工作区时用,别拿它当「清理一下」。
4. 两条命令默认等数据删完(`status:"done"`);超时回 `queued/running` + `jobId` 不是失败,`tablebi context` 的 `pending` 会显示「正在删除」。

## 命令清单

下面这段由 CLI 的命令注册表生成(`tablebi skill --print`),与 `--help` 的命令集合一致(CI 断言)。
`connect csv --platform <p>` 别和已实时同步的平台同名(会被拒收),用 `meta_ads_csv` 这类;`sync --slices geo,pages --days 180` 只补 GSC 页面 / 国家表的历史,不重拉明细。宿主更愿意走 MCP 时:`tablebi install --mcp` 注册内置的 `tablebi mcp`。

<!-- tablebi:commands:begin(`tablebi skill --print` 生成,别手改)-->
```
tablebi login [--api <url>]                           # 浏览器登录
tablebi logout                                        # 注销
tablebi context [--full] [--sample]                   # 开场 rehydrate:口径包 + 源 + 新鲜度 + 定义 + 看板 + 账号 + 命令签名
tablebi scoreboard [--window <days>]                  # 多产品增长记分牌:谁在涨 / 谁在跌
tablebi gsc <query|inspect|sitemaps>                  # GSC 直连 Google
tablebi ask <query>                                   # Ask:全功能只读 SQL
tablebi query [widget] [--kind <k>] [--view <v>] [--metrics <list>] [--by <dim>] [--window <w>] […]  # 问一个语法 widget
tablebi define [key] [value...] [--unset <key>]       # 工作区定义:品牌词 / 站点组 / 排除的站 / 属性显示名 / 阈值
tablebi connect <provider> [--site <url>] [--account <id>] [--days <n>] [--no-wait] [--file <path>] [--platform <p>]  # 连数据源:gsc|ga4|meta_ads|google_ads
tablebi sync <provider> [--site <url>] [--account <id>] [--days <n>] [--slices <list>] [--no-wait]  # 拉取一个已连源到 facts
tablebi disconnect <provider> [--site <url>] [--account <id>] [--platform <label>] [--yes] [--no-wait]  # 断开一个数据源并删除它的数据
tablebi workspace <delete>                            # 工作区管理
tablebi pin [--file <path>] [--title <t>] [--widget <w>]  # Pin:一步固化活看板
tablebi dashboard <list|show|create|spec|set-spec|publish|template|unpublish|annotate>  # 看板
tablebi install [--codex] [--dry-run] [--mcp]         # 把 SKILL 铺进 Claude Code / Codex
tablebi mcp                                           # 以 stdio MCP 服务器运行
tablebi skill [--print] [--check]                     # SKILL 的命令清单
tablebi update                                        # 升级 CLI 到最新版
tablebi pending                                       # 待处理:stale / 未连源
tablebi whoami | workspaces | status | schema | sources | sample | values | metrics  # 老命令,照常可用;将并入 context / query
```
所有命令都认 `-w, --workspace <ws>` 与 `--json`;子命令与完整参数以 `tablebi context` 返回的 `cli.commands` 为准。
<!-- tablebi:commands:end -->

`connect` 一条龙:弹浏览器授权(凭证加密存服务端,你碰不到密钥)→ 列目标(多个则输出 `{status:"choose_target",targets:[…]}`,带 `--site/--account` 再来)→ 同步 + 出看板。默认**等待完成**;超时输出 `{status:"in_progress"}`(≠失败)。

## 规则

- **数从引擎来,别编**:用 `query` / `ask` / `context` 的输出,不要自己猜数字。
- **能用 `query` 的别写 SQL**;写 SQL 时派生指标用宏(roas/ctr/cpa…),别手搓比值——口径只在一处定义。
- **常识用 `define` 沉淀**(品牌词、站点组、属性名、排除的站),别在每条 SQL 里硬编码。
- **按已连源选高度**:ROAS/CPA/花费类口径只对连了投放源的用户谈;GSC-only 用 raw 看涨跌,连提都别提 ROAS。
- **数据薄别硬找问题**:走冷启动陪跑,硬挤的"发现"毁信任。
- **被问"为什么和平台后台对不上"**:答案**引 `context` 的 `trust.caveats`**(按你连的平台组合自动列出的口径差异:归因窗口/源字段/去重差异),照它讲、别现编——如 Meta 转化=平台归因、GA4=last-click,天然不等;广告 revenue=平台归因价值非对账收入;GSC 点击≠GA4 会话。承认不可比处,别硬拗一致。caveats 没列到的才靠常识补,并说明是推断。
- **探索用 `query`,发布钉 v2 spec**:两边同一个编译器;`query` 返回的 `sql` 字段就是看板会跑的那条。
- **做完提示回 console**;**pin 完提议定时盯盘**(有异动才开口,别骚扰)。
- **平台是权威标签**:`connect csv --platform` 决定来源,不从列里猜。**已在实时同步的平台别拿它的标签导 CSV**(`meta_ads` / `google_ads` / `ga4` / `search_console`):重叠的日子会和实时数据相加、重复计数,所以服务端直接拒收,报错里给出该换的标签(如 `meta_ads_csv`)。换了标签只是能按 `platform` 分开看——不按 `platform` 过滤的合计仍会把同一账户同一天的两份都算进去,合计时二选一。
- **CSV 的花费按表头币种原值入库**(如「Amount spent (EUR)」),不换算:导入结果的 `currency` 就是它,`notes` 里的提醒照读给用户;和别的币种的平台一起合计前先讲清币种。
- 数据旧了先 `connect`/`sync` 再分析(看 `context` 的新鲜度与 `coverageThrough`)。
- **删除类命令(`disconnect` / `workspace delete`)永远先问人**:不带 `--yes` 的那次输出就是给人看的确认单;别自己加 `--yes`,也别为了「重来一遍」去删源再重连(重连会从头拉历史)。
- **用户面不出实现词**:没有 pod / warming / binding / parquet;渠道名用 `platform_label()` 或人话。
