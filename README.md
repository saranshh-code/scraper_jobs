# scraper_jobs

A [Claude Code](https://claude.com/claude-code) workspace for scraping job listings with the
[Firecrawl](https://firecrawl.dev) MCP server.

## What this actually is

**There is no scraper script here — and that's the design.** This repo contains no Python, no
Node, nothing to `run`. What it contains is *conventions*: a playbook that tells Claude how to
scrape properly, and a planning workflow for larger changes.

The scraping is done by Claude Code calling Firecrawl's tools directly. You open this folder,
paste a URL, and say what you want out of it. The rules in [FIRECRAWL.md](FIRECRAWL.md) make
sure the result is fresh, complete, and honestly reported instead of quietly cached or
half-finished.

So the mental model isn't "a program I run." It's "a well-briefed assistant I talk to."

## Setup

You need [Claude Code](https://claude.com/claude-code) and a Firecrawl API key
([get one here](https://www.firecrawl.dev/app/api-keys) — the free tier is 1000 credits/month).

```bash
cp .env.example .env          # then paste your key into .env
```

The key is needed in two places, because two different things consume it:

| File | Used by | Committed? |
|------|---------|-----------|
| `.env` | any scraper code you write later | no — gitignored |
| `.claude/settings.local.json` | the Firecrawl **MCP server** (Claude's tools) | no — gitignored |

`.mcp.json` wires up the server itself and *is* committed — it reads the key as
`${FIRECRAWL_API_KEY}`, so no secret lands in git. Note that Claude Code does **not** read
`.env` when expanding that variable; it comes from the `env` block in
`.claude/settings.local.json` or your shell.

Verify it works:

```bash
source .env
curl -s https://api.firecrawl.dev/v2/team/credit-usage \
  -H "Authorization: Bearer $FIRECRAWL_API_KEY"
```

A `success: true` with your remaining credits means you're set.

## Using it

Open the folder in Claude Code and ask in plain language:

> Scrape https://boards.greenhouse.io/example for open engineering roles — title, location, and salary.

Claude reads [FIRECRAWL.md](FIRECRAWL.md) automatically (via [CLAUDE.md](CLAUDE.md)) and follows
its procedure: work out what fields you want → pick the right tool → force a live fetch → handle
failures → report back with a preview table, a saved file, and the credits spent.

You don't have to know Firecrawl's tools. But if you want to steer, the useful ones are
`firecrawl_map` (list a site's URLs, cheap), `firecrawl_scrape` (one page → structured JSON),
`firecrawl_crawl` (many pages), and `firecrawl_search` (find URLs when you don't have them).

### What you get back

Results land in `data/<source>-<YYYY-MM-DD>.json`, wrapped in an envelope that records *how* the
data was obtained — not just the rows. From the existing Wellfound scrape:

```jsonc
{
  "source": "wellfound.com",
  "scraped_at": "2026-08-19",
  "query": "PM / product management intern jobs, global",
  "method": {
    "discovery": "firecrawl_map x4 (search terms: ...)",
    "verification": "firecrawl_scrape per job page, maxAge:0 (forced live fetch)",
    "liveness_test": "HTTP statusCode. 200 = live, 410 = delisted",
    "known_gap": "Wellfound role/location listing pages are client-rendered behind auth..."
  },
  "coverage": { "candidates_checked": 21, "live_200": 20, "delisted_410": 1 },
  "jobs": [ /* ... */ ]
}
```

That `method` / `coverage` block is the point. A scrape you can't audit later is a scrape you
can't trust — it tells you what was searched, what was verified live, and **what was missed**.

## The two rules worth knowing

**1. Cached data looks exactly like fresh data.** Firecrawl serves recently-indexed content by
default, and an HTTP `200` says nothing about freshness — a scrape here once returned content
cached ~24 hours earlier. For job listings that's the gap between open roles and filled ones. So
time-sensitive scrapes always pass `maxAge: 0`, and Claude reports `cacheState` back to you.

**2. Credits are finite.** A JSON-schema scrape of one page costs ~5 credits against a 1000/month
plan. Hence: use `map` to target before scraping, test a schema on one page before running it
across fifty, and expect a warning before anything expensive.

## Planning workflow

Small scrapes are just a conversation. Non-trivial work runs through three commands, each
handing off via a document in `docs/plans/<plan-id>/`:

| Command | Writes | Purpose |
|---------|--------|---------|
| `/plan` | `PRD.md` | Interviews you, then writes the requirements and work breakdown. **Never writes code.** |
| `/implement` | `IMPLEMENTATION.md` | Fans out `implementer` subagents across the work items, integrates, verifies. |
| `/review` | `REVIEW.md` | Read-only adversarial review through parallel lenses. Re-runs verification itself. |

The gates are deliberate: `/implement` refuses to start unless the PRD is marked `Approved` with
no `BLOCKING` questions, and asks before spawning agents. `/review` treats the implementation
report as a *claim*, not evidence. Details in [docs/plans/README.md](docs/plans/README.md).

Because each step hands off through a file, the chain survives across sessions — you can
`/implement` days after `/plan`, in fresh context, and nothing is lost.

## Layout

```
CLAUDE.md                     auto-loaded by Claude Code — the rules that always apply
FIRECRAWL.md                  the scraping playbook (tool choice, params, failures, reporting)
.mcp.json                     Firecrawl MCP server, scoped to this folder
.env / .env.example           API key for your own code
.claude/settings.local.json   API key for the MCP server (gitignored)
.claude/commands/             /plan, /implement, /review
.claude/agents/               implementer, plan-reviewer subagents
docs/plans/                   PRDs, implementation reports, review reports
data/                         scrape output, one file per source per day
```

## Troubleshooting

| Symptom | Try |
|---------|-----|
| Claude has no `firecrawl_*` tools | Restart Claude Code in this folder; check `.claude/settings.local.json` has the key |
| `401` from the API | Key is wrong or expired — regenerate at the Firecrawl dashboard |
| Scrape returns empty content | Page is likely JS-rendered — ask Claude to add `waitFor`, or turn off `onlyMainContent` |
| `403` / blocked | Ask Claude to escalate `proxy`: `auto` → `stealth` → `enhanced` |
| Data looks stale | Confirm `maxAge: 0` was used, and check `cacheState` in the response |

Fuller table in [FIRECRAWL.md](FIRECRAWL.md) §7.
