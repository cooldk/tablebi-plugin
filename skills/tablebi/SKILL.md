---
name: tablebi
min_cli: 0.3.0
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

每个会话先跑一次 `tablebi context --json`(口径 + 已连源 + 新鲜度 + 数据概览 + 命令签名)。数据为空就先 `tablebi connect …`。

命令若在 stderr 打印「正在准备你的数据环境…」是**正常等待**,CLI 会自己重试;只有最终返回 `status:"unavailable"`(退出码 3)才是真没好,照它的 `remediation` 稍后重跑同一条命令。

**多站 / 组合解读 —— 别把 N 个站说成 1 个:** 同一平台常连了多个站 / 账户(GSC 一个授权下 N 个验证域名)。`context` 的 `data.sites` 按 account 逐行给 `site`/`rows`/`dataThrough`/`ageDays`/`stale`,GSC 站另有 `clicks28d`(近 28 天站点总点击)与 `anonPct28d`(匿名占比 %:按搜索词拆开时看不到的那部分);GSC 站按 `clicks28d` 排在前,其余按行数,**只列前 20 个**;全部看 `tablebi sources` 或 `context --full`。

- 报「连了 N 个站」用 `data.siteCountByPlatform`(GSC 站与 GA4 属性分开数),别拿混在一起的 `siteCount` 当站数;**别**把平台级总行数 `data.rows` 当成某单站的行数,也**别**只报 `sample` 里碰巧出现的那个 account——那是采样,不是「唯一的站」。
- 平台级 `trust.freshness` 取各站 max,**会被一个跑得快的小站粉刷成新鲜**。数据真到哪天看 `coverageThrough`(所有主要站点都覆盖到的那天)与 `laggingSites`;站级陈旧计数看 `data.staleSites`。旧站单独 `sync`(GSC 用 `--site`)。

## 首次同步的 1–2 分钟:别让用户干等

首次连源拉数较慢。**别只甩一句"正在同步"就沉默**——用这段时间就着即将到来的数据先聊:确认连了哪个账户/站点、马上能看什么、你打算先帮他看哪条。说人话,**绝不暴露实现词**(没有 pod/warming/binding 这类词)。

- ✓「正在同步 plairmoi.com 最近的数据,大概一两分钟。等它好,我先帮你扫一眼各渠道 ROAS 有没有在烧钱的 —— 你最担心哪块?」
- ✗ 发一句"同步中…"然后干等

## 刚连完 / 同步完:就着真数据给钩子(激活的关键一跳)

**别甩"你可以问 ROAS / 看趋势"这种泛泛能力清单**——那是最弱的教育。先用一两个 `ask` 探一眼真实数字,再给 **2–3 个就着具体数、带钩子的下一步**。这一步直接决定新用户会不会走下去。

**先探一眼——按已连的源选探法,别一律打 ROAS**(ROAS/CPA 是**投放口径**,只对连了 Google Ads / Meta 的用户有意义):

- 有投放源 → 各渠道 ROAS 排序、异常花费:
  `tablebi ask --json "SELECT platform_label(platform) AS 渠道, round(roas(SUM(revenue),SUM(cost)),2) roas, round(SUM(cost)) cost FROM metrics GROUP BY 1 ORDER BY roas"`
- 只有 GSC(没投放)→ **别提 ROAS/花费**,看近 7 天 vs 前 7 天的页面/词涨跌(页面下钻 `search_console_pages_raw`、词下钻 `search_console_raw`,用 `FILTER (WHERE date > (SELECT max(date) FROM search_console_totals_raw) - INTERVAL 7 DAY)` 分两期)。
- 只有 GA4 → 渠道/来源流量的周环比大变动(`ga4_raw` 的 `sessions`/`conversions`)。

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

## Ask:两个高度

口径在 DuckDB 里焊成视图 + 宏,两边都查(`tablebi schema` 看维度/度量/宏,`tablebi sample` 看各表真实列名):

- **口径(统一可信)**:视图 `metrics`(友好列名,`conversion_value`→`revenue`)+ 宏 `roas` / `ctr` / `cpc` / `cpm` / `cpa` / `cvr` / `aov` / `avg_position` / `platform_label` / `ai_assistant`。派生比值是**聚合后**算,别手搓——用宏。
- **原生**:`<platform>_raw` 视图(`ga4_raw` / `meta_ads_raw` / GSC 的四张 …)——connect 时落的原生层,单平台字段(Meta 的 reach/frequency/cpm、Google Ads 带小数的 conversions 与 `cost_micros`、GSC 的 position…在这;广告两家的原生表也只到 campaign 级)。统一层只是可信的共同子集,要更细就下钻原生。统一裸表 `facts` 也在,用于跨渠道。`context` 的 `sql.views` 列出当前可查的视图。
- **GSC 四张原生表各管一类数,问什么查哪张**(别拿明细加总当总数):
  - 点击 / 曝光 / CTR / 平均排名、趋势、环比 → `metrics`(platform='search_console')或 `search_console_totals_raw`(**加 `search_type = 'web'`**;它还有 image / video / news / discover / googleNews 的行)。站点级,含匿名查询,= Search Console 后台总览。
  - 页面 → `search_console_pages_raw`(含匿名查询带来的点击;曝光与排名按页面计)。
  - 国家 / 设备 → `search_console_geo_raw`(`country` 是 ISO-3 小写如 `usa`,`device` 是 `DESKTOP` / `MOBILE` / `TABLET`)。
  - 搜索词、词×页、排名分布、机会词 → `search_console_raw`(明细,**不含匿名查询**:合计会小于总数,各站差多少看 `data.sites[].anonPct28d`)。
  - 平均排名一律 `avg_position(SUM(position*impressions), SUM(impressions))`,别用 plain avg。
  - 这四张表答不了的(词×国家/设备、图片或 Discover 明细、搜索外观、小时级、同步窗口之外的明细、收录、sitemap)→ 见下面「GSC 直连」。
- **GA4 三张原生表**:渠道 / 国家 → `ga4_raw`;具体来源 × 媒介(`google / organic`、`chatgpt.com / referral` …)→ `ga4_sources_raw`;AI 助手带来的访问 → `ga4_sources_raw` 加 `WHERE ai_assistant(source) IS NOT NULL`(宏按名单认 ChatGPT / Perplexity / Claude / Gemini / Copilot …),落到了哪些页 → `ga4_ai_landing_raw`(只含 AI 来源)。三张表的 `conversions` 都是 GA4 的 key events。

`--json` 给你解析,默认 CSV 给人看。**只读**:引擎已锁死(只能查这份数据,读不了别的文件/网络;超内存落盘、超时被杀)。一条 SELECT/WITH。

**追问顺序:按 `context` 的 `drill.next` 走。** 筛定一个维度之后,接下来还有信息量的维度是固定的
(定了站 → 问它靠哪些词 / 哪些页;定了词 → 问它落在哪个页;定了页 → 问它靠哪些词进来)。
**看板上的「点即筛」读的是同一张表** —— 用户点一行,那一维就被筛定、对应的榜单自动收起,
剩下的正是 `drill.next` 里那几个。按它追问,你给出的路径和用户自己点出来的是同一条;
自己另编一套,两边就会分叉。

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

**主路径:写 spec 文件,别在 shell 参数里转义 SQL。** 中文别名、子查询、`INTERVAL`、`"CTR%"` 在参数里要再转义一层,是最容易出错的地方。

```bash
cat > /tmp/spec.json <<'JSON'
{ "title": "搜索流量",
  "widgets": [
    { "title": "近 30 天概览", "chart": "scorecard",
      "sql": "SELECT SUM(impressions) AS 曝光, SUM(clicks) AS 点击, round(ctr(SUM(clicks),SUM(impressions))*100,2) AS \"CTR%\" FROM search_console_totals_raw WHERE search_type = 'web' AND date > (SELECT max(date) FROM search_console_totals_raw) - INTERVAL 30 DAY" },
    { "title": "每日趋势", "chart": "line",
      "sql": "SELECT date AS 日期, SUM(impressions) AS 曝光, SUM(clicks) AS 点击 FROM search_console_totals_raw WHERE search_type = 'web' GROUP BY date ORDER BY date" }
  ] }
JSON
tablebi pin --file /tmp/spec.json
```

单个 widget 的快捷方式:`tablebi pin --title "营销周报" --widget "每日趋势::line=SELECT date AS 日期, SUM(clicks) AS 点击 FROM metrics GROUP BY date ORDER BY date"`。

**看板是交互式 BI(ECharts),不只是表格。** 品牌深色主题 + 可交互图(hover 看精确值、拖滑块缩放、点图例开关线)。widget 加 `"chart"` 字段(或标题 `::<类型>` 后缀)选形态,缺省 = 表格。**支持的类型权威在 `context` 的 `chartKinds` 字段**(加了新类型自动出现在那),每种适合什么那里也写了。经验法则:

**概览数字 → `scorecard` 顶一排;趋势/逐日 → `line`;占比/份额 → `pie`;长名排行 → `hbar`,短名少类别 → `bar`;多指标明细留表格。** 建看板默认**先来一排 scorecard + 一两张趋势图**,别交一堆纯表格。渲染失败/数据不成图**自动回退表格,绝不空白**。

**Pin/publish 完不要只甩 token —— 把人引回 console。** 主动告诉用户:看板是**活的**(自刷新),公开只读链接可直接发给合伙人/客户;它也已**陈列在 console.tablebi.com → 看板**里,去那看全貌、管理、分享。对话负责创作,console 负责陈列与分享,做完就把人送回载体。

## 定时盯盘:pin 完主动提议「我每周替你看一眼」

没人天生记得回来看数——看板 pin 完,**主动问一句**:「要不要我每周一自动检查这张板,有大变动叫你?」用户点头就用你宿主的定时能力建周任务(Claude Code 用 scheduled tasks / cron):

1. 跑 `tablebi context --json`(顺带触发保鲜)+ 把看板核心指标和上周对比;多站 GSC 组合先 `tablebi scoreboard --json`(谁涨谁跌一眼出,省 N 条 ask)。
2. **有异动才开口**(某指标周环比超 ±20%、某源 stale):一句人话 + 数 + 下一步建议。
3. **怎么"叫人"**:定时任务里跑、用户不在对话 → 要有推送通道。**装了 clize** 就发一封简短邮件:一行异动 + 数 + **一句可粘回对话的话** + 看板链接(`clize email send`;发真人是硬闸,**先把草稿给用户看、他说发再带 `--confirm`**)。没 clize 就记下,下次对话主动说。用户就在对话里 → 直接说,别绕邮件。
4. **没异动就别刷存在感**(不发"一切正常"的水报告);数据薄的站例外——每周报一个成长小确幸。

这是 tablebi 形态独有的:别家托管 AI 只能发个死通知,你是能接着干活的 agent——发现异动当场下钻、当场改看板,邮件只是把人拉回对话的钩。默认频率**周**,用户嫌烦一句话就撤。

> 备注:这条是**用户自己 agent 层**的召回(BYO,零后端);产品侧另有常开后端 digest 邮件兜底,默认关、需另配,不在本 skill 职责内。

## 命令清单

**参数以 `tablebi context` 返回的 `cli.commands` 为准**(CLI 自报的签名,不会和 `--help` 漂移);下面只是常用面。

```
tablebi install / login / update       # 铺 skill / 浏览器授权登录 / 升级 CLI
tablebi context [--json] [--full]      # 开场 rehydrate(--full = 不截断站点列表)
tablebi schema | sources | pending | sample   # 自省:口径 / 已连源 / 待处理 / 样本行
tablebi connect <gsc|ga4|google_ads|meta_ads> [--site <url>|--account <id>]
tablebi connect csv --file <f> --platform <p>   # <p> 别和已实时同步的平台同名(会被拒收),用 meta_ads_csv 这类
tablebi sync <provider> --site <url>|--account <id> [--days <n>] [--slices <list>]   # GSC 可 --slices geo,pages --days 180 只补页面 / 国家表的历史,不重拉明细
tablebi ask "<SQL>" [--json]           # Ask(`sql` 是同义名)
tablebi pin --file <spec.json>         # Pin(主路径);或 --title + --widget "标题=SQL"
tablebi dashboard <list|show|spec|set-spec|publish|unpublish|annotate> <token>
tablebi scoreboard [--window <days>]   # 多站增长记分牌:谁在涨 / 谁在跌(GSC)
tablebi gsc query|inspect|sitemaps --site <站> …   # GSC 直连 Google(同步表答不了的才用,见上)
```

`connect` 一条龙:弹浏览器授权(凭证加密存服务端,你碰不到密钥)→ 列目标(多个则输出 `{status:"choose_target",targets:[…]}`,带 `--site/--account` 再来)→ 同步 + 出看板。默认**等待完成**;超时输出 `{status:"in_progress"}`(≠失败)。

## 规则

- **数从引擎来,别编**:用 `ask`/`context` 的输出,不要自己猜数字。
- **派生指标用宏**(roas/ctr/cpa…),别手搓比值——口径只在一处定义。
- **按已连源选高度**:ROAS/CPA/花费类口径只对连了投放源的用户谈;GSC-only 用 raw 看涨跌,连提都别提 ROAS。
- **数据薄别硬找问题**:走冷启动陪跑,硬挤的"发现"毁信任。
- **被问"为什么和平台后台对不上"**:答案**引 `context` 的 `trust.caveats`**(按你连的平台组合自动列出的口径差异:归因窗口/源字段/去重差异),照它讲、别现编——如 Meta 转化=平台归因、GA4=last-click,天然不等;广告 revenue=平台归因价值非对账收入;GSC 点击≠GA4 会话。承认不可比处,别硬拗一致。caveats 没列到的才靠常识补,并说明是推断。
- **探索给 SQL,发布留 spec**:Ask 用裸 SQL;Pin 钉的是 widget 定义。
- **做完提示回 console**;**pin 完提议定时盯盘**(有异动才开口,别骚扰)。
- **平台是权威标签**:`connect csv --platform` 决定来源,不从列里猜。**已在实时同步的平台别拿它的标签导 CSV**(`meta_ads` / `google_ads` / `ga4` / `search_console`):重叠的日子会和实时数据相加、重复计数,所以服务端直接拒收,报错里给出该换的标签(如 `meta_ads_csv`)。换了标签只是能按 `platform` 分开看——不按 `platform` 过滤的合计仍会把同一账户同一天的两份都算进去,合计时二选一。
- **CSV 的花费按表头币种原值入库**(如「Amount spent (EUR)」),不换算:导入结果的 `currency` 就是它,`notes` 里的提醒照读给用户;和别的币种的平台一起合计前先讲清币种。
- 数据旧了先 `connect`/`sync` 再分析(看 `context` 的新鲜度与 `coverageThrough`)。
- **用户面不出实现词**:没有 pod / warming / binding / parquet;渠道名用 `platform_label()` 或人话。
