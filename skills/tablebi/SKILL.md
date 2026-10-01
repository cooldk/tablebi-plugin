---
name: tablebi
min_cli: 0.4.2
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

# TableBI — the data backend for your Claude Code (Pipe → Ask → Pin)

tablebi is the noun (the home of the data and the dashboards); you are the verb (the hands that drive it). The brain is always you — tablebi hosts no LLM.
**Pipe** sources in and unify the metrics → **Ask** questions (at two altitudes) → **Pin** the answer as a public, read-only, self-refreshing dashboard URL.

## Language

Answer in the user's language. This skill is written in English, but many users write in Chinese — reply in whatever language they use. The CLI and the server answer in English by default (errors, hints, confirmations, `query` labels); they answer in Chinese when the CLI runs with `TABLEBI_LANG=zh` or a Chinese system locale — set `TABLEBI_LANG=zh` for a user who wants Chinese output. Data stays as it is in either language: dashboard titles people wrote, and the column names of the system dashboards and older CLIs (Chinese) — read it as data and translate it when you relay it.

## Rehydrate first

Start every session with `tablebi context --json` (metric packs `grammar.packs` + connected sources + freshness + workspace `definitions` + dashboards + account + command signatures). If there's no data yet, `tablebi connect …` first.

If a command prints "Preparing your data environment…" (「正在准备你的数据环境…」 in Chinese) on stderr, that's a **normal wait** — the CLI retries on its own. Only a final `status:"unavailable"` (exit code 3) means it really isn't ready; follow its `remediation` and rerun the same command later.

**Multiple sites / combined readings — don't call N sites one:** a platform often has several sites / accounts connected (N verified domains under one GSC authorization). In `context`, `data.sites` gives `site`/`rows`/`dataThrough`/`ageDays`/`stale` per account; GSC sites also carry `clicks28d` (the site's total clicks over the last 28 days) and `anonPct28d` (the anonymized share in %: the part you can't see once you split by query). GSC sites are ordered by `clicks28d`, the rest by rows, and **only the top 20 are listed**; see them all with `tablebi sources` or `context --full`.

- To say "N sites connected", use `data.siteCountByPlatform` (GSC sites and GA4 properties counted separately), not the combined `siteCount`; **don't** present the platform-wide `data.rows` as one site's rows, and **don't** report only the account that happens to appear in `sample` — that's a sample, not "the only site".
- The platform-level `trust.freshness` is the max across sites, so **one fast small site can paint the whole platform fresh**. How far the data really goes: `coverageThrough` (the day every major site covers) and `laggingSites`; the per-site staleness count is `data.staleSites`. Sync stale sites one by one (`--site` for GSC).

## The first sync's 1–2 minutes: don't leave the user waiting

The first pull after connecting is slow. **Don't just say "syncing" and go quiet** — use the time to talk about the data that's on its way: confirm which account / site got connected, what they'll be able to see right away, what you plan to look at first for them. Speak plainly and **never expose implementation words** (no pod / warming / binding).

- ✓ "Syncing plairmoi.com's recent data — a minute or two. Once it lands I'll check whether any channel is burning money on ROAS — which part worries you most?"
- ✗ Sending "Syncing…" and then waiting in silence

## Just connected / synced: hook them with real numbers (the key activation step)

**Don't hand over a generic capability list like "you can ask about ROAS / trends"** — that's the weakest kind of onboarding. First look at real numbers with one or two `query` calls, then offer **2–3 next steps anchored on specific numbers, each with a hook**. This step decides whether a new user keeps going.

**Look first — pick the probe by what's connected; don't default to ROAS** (ROAS/CPA are **paid-media metrics**; they only mean something to users with Google Ads / Meta connected):

- Paid sources → `tablebi query --kind table --view metrics --metrics cost,roas,conversions --by platform --json` (then look at campaigns for unusual spend).
- GSC only → **don't mention ROAS / spend**: `tablebi query --kind compare --view search_console --metrics clicks,impressions --by page --window last-7d --json` (pages rising / falling; `--by query` for queries).
- GA4 only → `tablebi query --kind compare --view ga4 --metrics sessions,conversions --by channel --window last-7d --json`.

**Then the hooks** (specific numbers > generic capabilities):

- ✓ "Brand Search ran at 0.9× ROAS this week and burned $1.2k — want me to pull it apart and see if that's wasted budget?"
- ✓ "Clicks on /best-summer-dresses are up 40% — want me to pin a dashboard that watches it?"
- ✗ "You can ask about ROAS by channel" (no numbers, no hook; for users with no paid media, don't even bring it up)

When the user says yes, **just do it**. Be honest: only claim what the data actually supports (a correlation that lines up in time — don't sell it as causation); when unsure, say "looks like it — want me to dig in and confirm?"

## Thin / almost empty data: a cold-start companion (don't force findings)

**Decide by `data.thin` in `context`** (true = no paid sources and under a few hundred of activity in the last 90 days; computed server-side — don't eyeball it). When data is thin, the hook pattern above **has nothing to hook on — don't squeeze one out**: a forced "finding" is a junk insight, and one is enough to wreck trust. Do this instead:

1. **Open honestly, and change the value proposition**: "Your site is still young, and there isn't enough data for real 'analysis' yet — but that's exactly why recording from day one is worth the most: three months from now you'll see the full growth curve, with evidence for which move lines up with which rise or dip."
2. **Connect everything that can be connected** (GA4 / historical CSV ledgers): the fuller the baseline, the better grounded later reads are.
3. **Start from the default dashboard**: the system already generated one (the one on the console home). Read its definition with `tablebi dashboard spec weekly`, edit it and `pin` it as your own; don't pin an identical copy (the default is overwritten on every sync).
4. **Offer a weekly check** (see "Weekly watch"): thin-data users stay for one small weekly win, not for depth.
5. Weekly hooks talk about **growth**, not problems: ✓ "3 queries made the top 50 this week, and /pricing got its first organic click" ✗ "Your ROAS…" (no paid media — never bring it up).

Once the data grows (paid sources, or GSC at a thousand-plus clicks a month), switch back to the hook pattern.

## Ask: `query` (grammar) first, `ask` (SQL) when it can't

`tablebi query` only says **what you want** (kind · view · metrics · by · window · compare · filter); the compiler picks the tables, anchors the dates, computes ratios on the same aggregation level and excludes brand terms. It returns rows + **the generated SQL** + the time window + trust — the number you ask about and the number you pin come from the same compiler.

```bash
tablebi query --kind kpi --view search_console --metrics clicks,impressions,ctr,avg_position --compare previous_period --json
tablebi query --kind breakdown --view search_console --metrics clicks,ctr --by query --filter non_brand --limit 20 --json
tablebi query --kind trend --view ga4 --metrics sessions --split-by ai_assistant --filter ai_only --window last-90d --grain week --json
tablebi query --kind compare --view metrics --metrics cost,roas --by platform --json
```

- kind: `kpi` one row of totals (≤ 4 metrics; add compare for period over period) · `trend` over time · `breakdown` what's biggest · `table` detail · `compare` this period vs the previous one.
- Which dimensions / metrics each pack has, and which tables this workspace actually has: `grammar.packs` in `context` (connected packs only). The compiler routes the GSC tables: totals go to the site-totals table, per-query to the detail table — relay the returned `caveats` (e.g. "detail excludes anonymized queries") along with the numbers.
- window defaults to `last-28d`, **anchored on the last day with data** (not today); weekly / monthly trends only draw complete buckets. filter: `dimension=value` (`*` wildcard, comma = OR), `<dimension>_not=` to exclude, `site_group=` / `exclude_sites` / `non_brand` / `brand` (GSC), `organic` / `ai_only` (GA4).
- Errors list the valid values — fix the call accordingly. What the grammar can't express (custom math, cross-table JOINs) goes to `ask` as SQL.

### The two altitudes of SQL (`ask`)

- **Unified**: the `metrics` view (cross-channel; the revenue column is `revenue`) + the macros `roas` / `ctr` / `cpc` / `cpm` / `cpa` / `cvr` / `aov` / `avg_position` / `platform_label` / `ai_assistant`. Ratios are computed **after aggregation** — use the macros, don't hand-roll them.
- **Native**: `<platform>_raw` (every field of one platform; the two ad platforms' native tables stop at campaign level, and Google Ads spend is `cost_micros`), plus the unified bare table `facts` (the revenue column is `conversion_value`). `sql.views` in `context` lists what you can query now; `tablebi context --sample` shows real column names.
- **The four GSC tables each own one kind of number** (never sum the detail to get a total): totals / trends → `search_console_totals_raw` (**add `search_type = 'web'`**; includes anonymized queries = the Search Console overview); pages → `search_console_pages_raw`; country / device → `search_console_geo_raw` (`country` is lowercase ISO-3, e.g. `usa`); queries → `search_console_raw` (**excludes anonymized queries**; how much each site is missing: `data.sites[].anonPct28d`). Average position is always `avg_position(SUM(position*impressions), SUM(impressions))`.
- **GA4**: channel / country → `ga4_raw`; source × medium → `ga4_sources_raw` (for AI assistant referrals add `WHERE ai_assistant(source) IS NOT NULL`); landing pages reached from AI → `ga4_ai_landing_raw`. `conversions` are always GA4 key events.

`--json` is for you to parse. **Read-only**: a single SELECT / WITH, with the engine locked down (no other files or network; long queries are killed).

**Follow `drill.next` in `context` for follow-up questions** (site fixed → ask about queries / pages; query fixed → pages; page fixed → queries). The dashboard's "click to filter" reads the same table: when the user clicks a row, that dimension gets fixed and what's left is exactly `drill.next` — follow it, and you and the user walk the same path.

## Definitions: capture the knowledge you'd otherwise repeat

`tablebi define` with no arguments lists the effective values (`defaulted` = keys still on their default). When the user states this kind of knowledge ("anything starting with tabl is a brand term", "these two properties are the same site", "leave the lokuma site out"), **define it on the spot** instead of hand-writing `NOT IN` / `VALUES` mappings into every SQL — the lint blocks those at publish time.

- Brand terms `tablebi define brand_terms "acme*, acme corp"` → the `non_brand` / `brand` filters work, and opportunity queries stop counting brand terms;
- Site groups `tablebi define site_group core "a.com, b.com"` → filter `site_group=core`; sites to leave out of the overview: `exclude_sites "old.com"`;
- GA4 property display names `property_alias "543715615 = tabledi.com"` (two IDs with the same name = merged into one row); site ↔ property `site_property tabledi.com "543715615"` (when a per-site dashboard picked the wrong property);
- Thresholds `striking_distance "4-15, 5"` (opportunity queries) and `movers "min_clicks=5"` (the movers list); conversion definitions `ga4_conversion_events "purchase, sign_up"` / `meta_conversion_actions "purchase, lead"` (effective from the next sync).
- Dashboard page language `language`: **new workspaces default to `en`** (public dashboards show titles, headers, badges and footers in English); when the user talks to you in Chinese and the dashboards are for Chinese readers, run `tablebi define language zh`. It only sets the dashboard pages' language (`query` output follows the CLI's language instead); templates from `dashboard template` carry it as `defaults.lang`.

The system default dashboards (workspace overview + one per site) build their sections from these definitions; changing one rebuilds them in the background. Remove one with `--unset <key>`.

## Direct GSC: only for what the synced tables can't answer

`tablebi gsc query|inspect|sitemaps` has the server call Google for you (credentials never leave the server), and only for sites **connected to this workspace**.

1. **`ask` first**: totals / trends → `metrics` or `search_console_totals_raw` (`search_type = 'web'`); pages → `search_console_pages_raw`; queries → `search_console_raw`; country / device → `search_console_geo_raw`.
2. **Use `tablebi gsc query` only when the synced tables can't answer**: query × country/device, image or Discover detail, search appearance, hourly data, detail outside the sync window.
3. **Don't use the direct route to copy numbers the synced tables already have**: it wastes Google quota and can slow down that day's sync.
4. **`tablebi gsc inspect` is for a few key URLs** (≤ 20 per call, 200 per site per day) — don't scan a whole site with it.

```bash
tablebi gsc query --site chatdiagram.com --filter "country = usa" --filter "device = MOBILE"   # queries from mobile users in the US
tablebi gsc query --site nichelogo.app --type image --dims page                              # pages that get image-search traffic
tablebi gsc query --site chatdiagram.com --dims query,page --filter "page contains /tool/"   # queries for one directory
tablebi gsc query --site chatdiagram.com --dims hour --days 1                                # today, by hour
tablebi gsc inspect --site chatdiagram.com https://chatdiagram.com/pricing                   # indexed? canonical right?
tablebi gsc sitemaps --site chatdiagram.com                                                  # sitemaps fetched? any errors?
```

- Defaults: `--dims query`, `--type web`, the last 28 days (through yesterday; includes today when grouping by hour), `--limit 1000` (max 25000; page with `--start-row`). Multiple `--filter`s are ANDed; the ops are `= != contains !contains regex !regex`, and regex is RE2. `country` is lowercase ISO-3 (`usa`); `device` is `DESKTOP` / `MOBILE` / `TABLET`.
- **Two mistakes Google won't flag — it just returns nothing**: hard-coding the host in a page filter (pages often live on www or a subdomain; filter a directory with `page contains /dir/`), and invalid regex syntax. When a result is empty, read `notes` first.
- The returned `notes` are metric caveats (grouping by query excludes anonymized queries, from which day values are preliminary, whether the result was truncated) — relay them with the numbers; `truncated: true` means there are more rows. `request` echoes the window actually queried.
- Limits: grouping or filtering by both query and page covers at most 93 days per call; data goes back 16 months at most; 20 calls per minute per workspace; identical requests hit the cache (`cached: true`).
- Errors: a 400 carries Google's own words (usually an invalid dimension combination) — fix the arguments as it says; 403 → ask the user to run `tablebi connect gsc` again; 429 → wait as `remediation` says (Google's quota takes 15 minutes, the per-minute rate limit only seconds) and don't repeat the same query.

## Pin: make it a live dashboard

Pin a set of queries as a public read-only URL (recomputed from **current data** every time it's opened — not a dead screenshot).

**Main path: a v2 spec file + `dashboard publish --file`.** Widgets use the same grammar as `query`; the chart type follows the kind (kpi → scorecard, trend → line, breakdown → horizontal bars; override with `"chart": "bar" | "pie"`), the time window goes into each tile's badge, and the title is optional (generated from the grammar, ≤ 40 characters).

```bash
tablebi dashboard template seo-overview > /tmp/board.json   # a template for your connected sources; no argument lists them all
# edit /tmp/board.json: drop sections, add sections, change window / filter / limit
tablebi dashboard publish --file /tmp/board.json --dry-run  # compiles + really runs + lints every widget
tablebi dashboard publish --file /tmp/board.json            # create and publish; to change an existing one: dashboard publish <token> --file …
```

Shape: `{ "version": 2, "title": "…", "defaults": { "view": "search_console", "window": "last-28d" }, "widgets": [ { "kind": "kpi", "metrics": ["clicks", "ctr"], "compare": "previous_period" }, { "kind": "breakdown", "metrics": ["clicks"], "by": "page", "limit": 15 } ] }`.
≤ 12 widgets per dashboard (hard cap 16); several dashboards of the same shape (one per site / property) become one with `split_by`. By default a dashboard **opens with a row of kpis + one or two trends** — don't hand over a pile of plain tables.

**SQL is the escape hatch**: `{ "kind": "sql", "sql": "…", "title": "…", "chart": "line" }`. The page marks it as a custom query, and in a v2 dashboard it has to pass the lint (no inline VALUES mapping tables, no `NOT IN` with more than 3 items, no `current_date` / `now()`, no GROUP BY without a LIMIT). The old format `{ title, widgets: [{ title, sql, chart }] }` and `tablebi pin --file` still work (v1 is always read; the lint only warns); the authoritative list of chart types is `chartKinds` in `context`.

**After a pin / publish, don't just hand over a token — send the user back to the console.** Tell them the dashboard is **live** (it refreshes itself), and the public read-only link can go straight to a partner or client; it's also **on display at console.tablebi.com → Dashboards**, where they can see the whole picture, manage and share it. The conversation creates; the console displays and shares — when you're done, send them back there.

## Weekly watch: after a pin, offer "I'll check it for you every week"

Nobody naturally remembers to come back and look — after pinning a dashboard, **ask**: "Want me to check this dashboard every Monday and flag anything big?" If they say yes, create a weekly task with your host's scheduling (Claude Code: scheduled tasks / cron):

1. Run `tablebi context --json` (which also triggers a freshness pass) and compare the dashboard's core metrics with last week; for multi-site GSC setups, start with `tablebi scoreboard --json` (who's up and who's down at a glance, saving N `ask` calls).
2. **Speak up only when something moved** (a metric beyond ±20% week over week, a stale source): one plain sentence + the number + a suggested next step.
3. **How to flag them**: the scheduled task runs while the user isn't in the conversation, so you need a push channel. **If clize is installed**, send a short email: one line on the change + the number + **one sentence they can paste back into the conversation** + the dashboard link (`clize email send`; emailing a real person is a hard gate — **show the user the draft first, and add `--confirm` only after they say send**). Without clize, note it and bring it up in the next conversation. If the user is right there in the conversation, just tell them — don't route it through email.
4. **Nothing moved → don't show up** (no "all good" filler reports); thin-data sites are the exception — report one small growth win each week.

This is unique to the way TableBI is built: other hosted AI can only send a dead notification, while you're an agent that can keep working — drill in the moment you see a change, fix the dashboard on the spot; the email is just the hook that brings the person back to the conversation. The default cadence is **weekly**; if the user finds it noisy, one sentence turns it off.

> Note: this is recall at the **user's own agent layer** (BYO, zero backend); the product also has an always-on backend digest email as a fallback — off by default and configured separately, and not this skill's job.

## Disconnect a source / delete a workspace (irreversible)

When the user says "disconnect / remove this site / this account", "delete my data" or "delete this workspace":

1. First run `tablebi disconnect <provider> --site|--account|--platform <target> --json` **without** `--yes`: it **deletes nothing** and only returns `{status:"confirm_required", effect, next}` (exit code 2). Tell the user exactly what `effect` says, and only after **explicit consent** run the command in `next` (it carries `--yes`).
2. Disconnecting = stop syncing + delete all the data synced from this source; if it's the last source of its type, the authorization is deleted too (for revoking it on the platform side, pass the returned `note` on to the user). For a CSV, use the `--platform` label it was imported with.
3. Deleting a whole workspace: `tablebi workspace delete <name> --confirm <name>` — its data, dashboards (their public links stop working) and authorizations are all deleted. Use it only when the user **explicitly** asks to delete the whole workspace — never as a "cleanup".
4. Both commands wait for the data to be deleted by default (`status:"done"`); a timeout returns `queued/running` + a `jobId`, which is not a failure — `pending` in `tablebi context` shows it as being deleted.

## Command list

The block below is generated from the CLI's command registry (`tablebi skill --print`) and matches the command set of `--help` (asserted in CI).
Don't give `connect csv --platform <p>` the name of a platform that already syncs live (it gets rejected) — use something like `meta_ads_csv`; `sync --slices geo,pages --days 180` backfills only the GSC page / country tables without re-pulling the detail. When the host prefers MCP: `tablebi install --mcp` registers the built-in `tablebi mcp`.

<!-- tablebi:commands:begin (generated by `tablebi skill --print`; do not edit by hand) -->
```
tablebi login [--api <url>]                           # Sign in in the browser
tablebi logout                                        # Sign out
tablebi context [--full] [--sample]                   # Session rehydrate: metric packs, sources, freshness, definitions, dashboards, account, commands
tablebi scoreboard [--window <days>]                  # Multi-site growth scoreboard: who is rising / falling
tablebi gsc <query|inspect|sitemaps>                  # Query Google Search Console directly
tablebi ask <query>                                   # Ask: full read-only SQL
tablebi query [widget] [--kind <k>] [--view <v>] [--metrics <list>] [--by <dim>] [--window <w>] […]  # Ask one grammar widget
tablebi define [key] [value...] [--unset <key>]       # Workspace definitions: brand terms / site groups / excluded sites / property names / thresholds
tablebi connect <provider> [--site <url>] [--account <id>] [--days <n>] [--no-wait] [--file <path>] [--platform <p>]  # Connect a source: gsc|ga4|meta_ads|google_ads
tablebi sync <provider> [--site <url>] [--account <id>] [--days <n>] [--slices <list>] [--no-wait]  # Pull a connected source into facts
tablebi disconnect <provider> [--site <url>] [--account <id>] [--platform <label>] [--yes] [--no-wait]  # Disconnect a source and delete its data
tablebi workspace <delete>                            # Workspace management
tablebi pin [--file <path>] [--title <t>] [--widget <w>]  # Pin: a live dashboard in one step
tablebi dashboard <list|show|create|spec|set-spec|publish|template|unpublish|annotate>  # Dashboards
tablebi install [--codex] [--dry-run] [--mcp]         # Install the SKILL into Claude Code / Codex
tablebi mcp                                           # Run as a stdio MCP server
tablebi skill [--print] [--check]                     # The SKILL's command list
tablebi update                                        # Upgrade the CLI to the latest version
tablebi pending                                       # Pending: stale / unconnected sources
tablebi whoami | workspaces | status | schema | sources | sample | values | metrics  # legacy, still work; merging into context / query
```
Every command accepts `-w, --workspace <ws>` and `--json`; for subcommands and full flags, `cli.commands` from `tablebi context` is authoritative.
<!-- tablebi:commands:end -->

`connect` does it all: authorize in the browser (credentials are encrypted and stored server-side — you never touch the keys) → list targets (if there are several it outputs `{status:"choose_target",targets:[…]}`; run it again with `--site/--account`) → sync + build dashboards. It **waits for completion** by default; a timeout outputs `{status:"in_progress"}` (≠ failure).

## Rules

- **Numbers come from the engine — never make them up**: use the output of `query` / `ask` / `context`; don't guess figures.
- **Don't write SQL when `query` can do it**; when you do write SQL, derived metrics use the macros (roas/ctr/cpa…) — don't hand-roll ratios; each definition lives in one place.
- **Capture common knowledge with `define`** (brand terms, site groups, property names, excluded sites) instead of hard-coding it into every SQL.
- **Pick the altitude by what's connected**: ROAS/CPA/spend-type metrics only for users with paid sources; for GSC-only users, look at rises and falls in the raw tables and don't even mention ROAS.
- **Thin data: don't force findings** — take the cold-start route; squeezed-out "findings" destroy trust.
- **Asked "why doesn't this match the platform's UI?"**: answer **from `trust.caveats` in `context`** (the definitional differences listed automatically for the platforms you've connected: attribution windows / source fields / deduplication) — follow it, don't improvise. E.g. Meta conversions = platform attribution while GA4 = last-click, so they naturally differ; ad revenue = platform-attributed value, not reconciled revenue; GSC clicks ≠ GA4 sessions. Admit what isn't comparable instead of forcing a match. Fall back on general knowledge only for what the caveats don't cover, and say it's an inference.
- **Explore with `query`, publish with a v2 spec**: both run the same compiler; the `sql` field returned by `query` is exactly what the dashboard will run.
- **When done, point back to the console**; **after a pin, offer the weekly watch** (speak up only when something moved; don't pester).
- **The platform is the authoritative label**: `connect csv --platform` decides the source; it's never guessed from the columns. **Never import a CSV under the label of a platform that already syncs live** (`meta_ads` / `google_ads` / `ga4` / `search_console`): the overlapping days would add up with the live data and double count, so the server rejects it and the error names the label to use instead (e.g. `meta_ads_csv`). A different label only lets you view them separately by `platform` — totals that don't filter by `platform` still count both copies of the same account's same day, so pick one when totaling.
- **CSV spend is stored at face value in the header's currency** (e.g. "Amount spent (EUR)"), not converted: the import result's `currency` is that currency, and read the reminders in `notes` to the user; before totaling it with platforms in another currency, spell out the currencies.
- If the data is stale, `connect`/`sync` before analyzing (see freshness and `coverageThrough` in `context`).
- **Deletion commands (`disconnect` / `workspace delete`) always ask a person first**: the output without `--yes` is the confirmation sheet for a human; never add `--yes` yourself, and never delete and reconnect a source to "start over" (reconnecting pulls the whole history again).
- **No implementation words in front of the user**: no pod / warming / binding / parquet; name channels with `platform_label()` or in plain words.
