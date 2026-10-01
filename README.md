# TableBI for Claude Code

The data backend for your Claude Code. Connect your marketing sources once, ask questions across
them in SQL, and pin the answer as a live dashboard URL that refreshes itself.

- **Pipe** — Google Search Console, Google Analytics 4 and Google Ads over OAuth; Meta Ads in beta;
  any CSV you upload.
- **Ask** — two altitudes: unified cross-channel metrics (`roas()`, `cpa()`, `ctr()` mean the same
  thing for every platform) or each platform's own raw table, in plain read-only SQL.
- **Pin** — turn a set of queries into a public, read-only dashboard link that stays current.

**See it before you install anything:** a [live demo dashboard on real data](https://tablebi.com/demo/?utm_source=github&utm_medium=plugin_readme&utm_campaign=tablebi-plugin)
— no account needed.

This plugin is the skill that teaches your agent the workflow. The work is done by the `tablebi`
command-line tool, which talks to the hosted service at [tablebi.com](https://tablebi.com). There
is no language model on our side: the reasoning happens in your own Claude Code.

## Install

Install the plugin:

```
/plugin marketplace add cooldk/tablebi-plugin
/plugin install tablebi@tablebi
```

Then install the CLI the skill drives and sign in:

```
npm i -g @tablebi/cli
tablebi login
```

`tablebi login` opens your browser to sign in with Google. When you connect Search Console, GA4 or
Google Ads, Google first shows a "This app isn't verified" screen while TableBI's Google review is
pending: choose **Advanced**, then continue to console.tablebi.com. The Search Console and GA4
permissions are read-only.

Prefer not to use plugins? `npm i -g @tablebi/cli && tablebi install` puts the same skill in
`~/.claude/skills` and keeps it up to date. Use one route or the other, not both, or Claude Code
will load the skill twice.

## Try it

Ask your agent, in your own words:

- "Connect my Search Console for example.com and tell me which pages are losing clicks."
- "Which queries sit just off page one?"
- "Compare ROAS by channel for the last 28 days." (needs an ad account connected)
- "Pin that as a dashboard and give me the link."
- "Is the data fresh?"

## What works today

| Source | Status |
| --- | --- |
| Google Search Console | Live — site totals, pages, queries, country × device |
| Google Analytics 4 | Live — sessions, users, conversions and revenue by channel and country |
| Google Ads | Live — campaign level |
| Meta Ads | Beta — campaign level; until Meta's App Review, only people added to our Meta app as test users can connect |
| CSV | Live — upload any export with `tablebi connect csv --file <f> --platform <label>` |

Not included: white-label reports, scheduled PDF or email reports, alerting, rank tracking. TableBI
is a CLI first; since 0.4.0 it also carries a local MCP server (`tablebi install --mcp` registers
`tablebi mcp`), and there is no hosted MCP endpoint. The free tier covers one source and one live dashboard; see
[pricing](https://tablebi.com/pricing.html).

## What runs, and where data goes

The plugin itself is one skill (`skills/tablebi/SKILL.md`): instructions, no scripts, hooks or MCP
servers. The skill tells your agent to use the `tablebi` command-line tool, which you install from npm
(`@tablebi/cli`). That tool talks to:

- `console.tablebi.com` — sign-in, connecting sources, and once a day a check for a newer copy of this
  skill (`/skill/tablebi/meta.json`);
- `<your-workspace>.tablebi.com` — your queries and dashboards;
- `registry.npmjs.org` — at most once a day, to tell you when a newer CLI is out.

It sends no analytics about you or your prompts. The OAuth tokens for Search Console, GA4, Google Ads and
Meta Ads are held on TableBI's server, never on your machine.

## Data and privacy

Connected data is stored in your workspace on the hosted service. OAuth tokens are encrypted at
rest, and your marketing data is never sent to a third-party model — your agent reads query results
through the CLI. Dashboards you publish are public read-only links; unpublishing takes them down.

- Privacy policy: https://tablebi.com/privacy/
- Terms: https://tablebi.com/terms/
- Support: support@tablebi.com

## License

The skill and plugin files in this repository are MIT-licensed. The TableBI service is operated by
tablize tech llc.
