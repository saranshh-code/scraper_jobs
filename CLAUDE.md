# scraper_jobs

Job-listing scraping built on the Firecrawl MCP server.

## Scraping rule

**Whenever the user gives you a URL, a website, or asks for anything scraped, read [FIRECRAWL.md](FIRECRAWL.md) first and follow its procedure** — intake (§1) → tool choice (§2) → params (§3) → failure handling (§7) → report (§8).

Non-negotiables from that file:

- Time-sensitive data (jobs, prices, availability) → `maxAge: 0`, and report `cacheState` / `cachedAt`. A `200` does not mean fresh.
- Bound every `firecrawl_crawl` with `limit`. Test schemas on one page before fanning out.
- Full data to `data/<source>-<YYYY-MM-DD>.json`; preview table in chat, never the raw blob.
- Report coverage and credits used. Never present a partial result as complete.
- `firecrawl_interact` and `firecrawl_monitor_create` hit live sites / schedule real activity — confirm first.

## Layout

- [FIRECRAWL.md](FIRECRAWL.md) — scraping playbook
- [.mcp.json](.mcp.json) — Firecrawl server, project-scoped
- `.env` / `.claude/settings.local.json` — API key, both gitignored
- `data/` — scrape output
