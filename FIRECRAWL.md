# Firecrawl Cheat Sheet

Operating manual for scraping in this project. **Claude: read this before running any `firecrawl_*` tool.**
When the user drops one or more URLs, walk §1 → §2 → §3 → §7 and report using §8.

---

## 1. Intake — settle these before calling anything

| Question | Default if the user didn't say |
|---|---|
| **What fields do they want out?** | Ask, or infer from the page type. Never dump raw markdown when they asked for data. |
| **One page, a list of pages, or a whole site?** | Whatever they literally gave. Don't widen scope to a crawl unattended. |
| **How fresh must it be?** | Job/price/stock data → `maxAge: 0`. Docs/reference → allow cache. |
| **Where does output go?** | `data/<source>-<YYYY-MM-DD>.json`. Print a preview table in chat, not the whole payload. |

If the answer to "what fields" is genuinely obvious (a job board → title/company/location/url/salary/posted), just proceed with a schema and say what you assumed.

---

## 2. Pick the tool

```
Known URL, want its content ─────────────► firecrawl_scrape
Known URL, want typed fields ────────────► firecrawl_scrape + formats:["json"]
"What pages exist on this site?" ────────► firecrawl_map          (no page bodies, cheap)
Many pages under one site ───────────────► firecrawl_crawl        (bounded — see §4)
Don't know the URL yet ──────────────────► firecrawl_search
Coding/library/error question ───────────► firecrawl_developer_search
Local PDF / DOCX / XLSX / RTF / ODT ─────► firecrawl_parse        (remote URLs go to scrape)
Needs clicks, login, form fill ──────────► firecrawl_interact  ⚠ acts on the live site
Open-ended multi-source research ────────► firecrawl_agent → firecrawl_agent_status
Watch a page for changes over time ──────► firecrawl_monitor_create
```

**The default combo for an unfamiliar site:** `firecrawl_map` (find the right URLs, cheap) → `firecrawl_scrape` with a JSON schema on the ones that matter. Don't reach for `crawl` just because there are several pages.

---

## 3. `firecrawl_scrape` — the workhorse

```jsonc
{
  "url": "https://example.com/jobs",
  "formats": ["json"],          // markdown | html | rawHtml | links | summary |
                                // screenshot | branding | changeTracking | query | audio
  "onlyMainContent": true,      // strips nav/footer/ads — on by default, turn OFF if content goes missing
  "maxAge": 0,                  // 0 = force live fetch. Omit to allow cache reuse.
  "jsonOptions": {
    "prompt": "Extract every job listing on the page.",
    "schema": { "type": "object", "properties": { "...": {} } }
  }
}
```

Other params worth knowing: `waitFor` (ms, for JS render), `includeTags` / `excludeTags` (CSS selectors), `proxy` (`basic`→`auto`→`stealth`→`enhanced`), `location: {country, languages}`, `mobile`, `removeBase64Images`, `redactPII`, `parsers: ["pdf"]`, `pdfOptions.maxPages`, `screenshotOptions.fullPage`.

### The caching gotcha — read this one
Firecrawl serves **recently indexed content by default**, and the reuse window varies by domain. A verified example from this project: a scrape of `news.ycombinator.com/jobs` returned `cacheState: "hit"` with content cached ~24h earlier. A `200` response does **not** mean the data is current.

→ **For job listings, prices, availability, or anything time-sensitive, always set `maxAge: 0`.**
→ Always check `metadata.cacheState` and `metadata.cachedAt` in the response and report staleness to the user.

---

## 4. `firecrawl_crawl` — bound it, always

Crawls return large payloads and burn credits fast. Never call it without limits.

```jsonc
{
  "url": "https://example.com",
  "limit": 50,                        // ALWAYS set this
  "maxDiscoveryDepth": 2,
  "includePaths": ["^/careers/.*"],   // regex against path
  "excludePaths": ["^/blog/.*"],
  "delay": 1,                         // seconds — be polite
  "maxConcurrency": 2,
  "sitemap": "include",               // include | skip | only
  "scrapeOptions": { "formats": ["json"], "onlyMainContent": true, "jsonOptions": { } }
}
```

`allowSubdomains`, `allowExternalLinks`, `crawlEntireDomain`, `deduplicateSimilarURLs`, `ignoreQueryParameters` are available. `crawl` polls to completion itself; use `firecrawl_check_crawl_status({id})` to re-read a job later.

---

## 5. `firecrawl_search`

```jsonc
{
  "query": "site:lever.co backend engineer remote",
  "limit": 10,
  "sources": [{"type": "web"}],            // web | news | images
  "categories": ["developer"],             // github | research | pdf | developer
  "includeDomains": ["boards.greenhouse.io"],   // mutually exclusive with excludeDomains
  "scrapeOptions": { "formats": ["markdown"] }
}
```

Operators: `"exact phrase"`, `-exclude`, `site:`, `inurl:`, `intitle:`, `related:`.

⚠ Pages fetched via `scrapeOptions` here use a **fixed reuse window and ignore `maxAge`**. Need guaranteed-live content? Search first, then `firecrawl_scrape` the hits separately.

---

## 6. Credits — spend them deliberately

Verified on this account: plain JSON-schema scrape of one page = **5 credits**. Plan is 1000/month.

- Check balance: `curl -s https://api.firecrawl.dev/v2/team/credit-usage -H "Authorization: Bearer $FIRECRAWL_API_KEY"`
- `map` is cheap (URLs only, no bodies) — use it to target before scraping.
- Test a schema on **one** page before running it across fifty.
- Warn the user before any call likely to exceed ~100 credits.

---

## 7. When it doesn't work

| Symptom | Fix, in order |
|---|---|
| Empty / truncated content | `onlyMainContent: false` → add `waitFor: 3000` → try `formats: ["rawHtml"]` |
| Content loads via JS | `waitFor: 2000–5000`, then `firecrawl_interact` if it needs a click |
| 403 / blocked / bot wall | `proxy: "auto"` → `"stealth"` → `"enhanced"` |
| Stale data | `maxAge: 0`, then confirm `cacheState` in the response |
| Wrong region content | `location: {"country": "IN"}` |
| Schema returns nulls | Loosen the schema, sharpen `jsonOptions.prompt`, verify the field is actually on the page |
| Pagination | `map` with `search` to enumerate pages, or find the `?page=N` pattern and scrape the range |
| Infinite scroll | `firecrawl_interact` with a scroll prompt |

**Rule: never silently return a partial result.** If 6 of 40 pages failed, say which and why.

---

## 8. Reporting back

1. **Preview table in chat** — first 3–5 rows, not the raw JSON blob.
2. **Full data to a file** under `data/`, and give the path as a link.
3. **State freshness** — live fetch, or cached from when.
4. **State coverage** — "40 of 40 pages", or exactly what was missed.
5. **State cost** — credits used, from `metadata.creditsUsed`.

---

## 9. Job-scraping playbook (this project)

Standard schema — extend per board:

```json
{
  "type": "object",
  "properties": {
    "jobs": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "title":       {"type": "string"},
          "company":     {"type": "string"},
          "location":    {"type": "string"},
          "remote":      {"type": "boolean"},
          "salary":      {"type": "string"},
          "posted_date": {"type": "string"},
          "url":         {"type": "string"},
          "description": {"type": "string"}
        }
      }
    }
  }
}
```

Flow: `map` the careers domain with `search: "job"` → scrape the index page with the schema and `maxAge: 0` → scrape individual postings only if the index lacks descriptions → dedupe on `url` → write `data/<board>-<date>.json`.

For recurring runs, prefer `firecrawl_monitor_create` (`page`/`pages` + a plain-language `goal`, optional `scheduleText`, `email`, `webhookUrl`, `includeDiffs`) over re-crawling — it diffs against the previous check instead of re-reading everything.

---

## 10. Setup facts

- Key lives in [.env](.env) (for scraper code) and [.claude/settings.local.json](.claude/settings.local.json) (for the MCP server). Both gitignored.
- Server config: [.mcp.json](.mcp.json), project-scoped to this folder.
- A duplicate user-scope entry exists in `~/.claude.json` and currently takes precedence.
- `firecrawl_interact` and `firecrawl_monitor_create` cause **real side effects** on live sites / schedule real network activity. Confirm with the user first.
