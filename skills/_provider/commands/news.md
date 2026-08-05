---
description: Digital asset news search, sentiment filtering, and daily digests
argument-hint: "[keyword] [--category defi-protocols|regulatory-news|...] [--sentiment positive|negative|neutral] [--days N] [--daily]"
allowed-tools:
  - mcp__Lukka-News__news___getDailySummaries
  - mcp__Lukka-News__news___getNews
---

# /news

> See [CONNECTORS.md](../CONNECTORS.md) for tool details.

## Usage

/news $ARGUMENTS

Parse $ARGUMENTS:
- Bare text = keyword search (e.g. `/news bitcoin ETF`)
- `--category` or `--cat` = primaryCategory filter
- `--sentiment` or `--sent` = sentiment filter (positive/negative/neutral/mixed)
- `--days N` = look back N days (default 7)
- `--daily` = show daily summaries instead of articles
- `--limit N` = max results (default 20, max 500)
- No arguments = last 7 days of daily summaries

## Flow

### Mode A: Daily summaries (no args, or `--daily`)

1. `mcp__Lukka-News__news___getDailySummaries({ limit: 7 })` [~~news]
   - Adjust `limit` if `--days N` specified
   - Monday summaries cover Saturday + Sunday (weekend digest)
   - Summaries available from 2026-04-08

### Mode B: News search (keyword or filters provided)

1. Compute date range from `--days N` (default 7):
   - `publishDateFrom`: `{N_days_ago}T00:00:00Z`
   - `publishDateTo`: `{today}T23:59:59Z`
   - **CRITICAL: Always pass full ISO 8601 with time+zone (e.g. `2026-05-01T00:00:00Z`).**

2. `mcp__Lukka-News__news___getNews({ keyword, primaryCategory, sentiment, publishDateFrom, publishDateTo, qualityFrom: 50, sortBy: "date", sortOrder: "desc", limit })` [~~news]

### Mode C: Trending (user asks "what's trending" or "top stories")

1. `mcp__Lukka-News__news___getNews({ relevanceFrom: 75, sortBy: "relevanceScore", sortOrder: "desc", publishDateFrom: "{7d_ago}T00:00:00Z", publishDateTo: "{now}T23:59:59Z", limit: 15 })` [~~news]

## Output

### Mode A: Daily Summaries

**Digital Asset News Digest - Last {N} Days**

| Date | Sentiment | Articles | Key Themes |
|------|-----------|----------|------------|
| 2026-05-16 | Positive | 142 | {keyThemes[0..2]} |

Per-day detail (if few days):
- **{date}**: {summaryText excerpt}
  - Notable: {notableEvents[0..2] titles}
  - Top assets: {topAssets[0..2]}

### Mode B/C: News Articles

**Digital Asset News - "{keyword}" ({N} results)**

| # | Date | Title | Source | Sentiment | Category | Quality |
|---|------|-------|--------|-----------|----------|---------|
| 1 | 2026-05-16 | {title} | {entityName} | positive | defi-protocols | 85 |

For small result sets (<=5), expand each article:
- **{title}**
  - {publishDateTime} | {entityName} | {sentiment}
  - {snippet}
  - Quality: {qualityScoreTotal}/100 | Relevance: {relevanceScoreTotal}/100

## Valid Categories

Valid `primaryCategory` / `newsCategory` values (AI-tagged, guided taxonomy - not a fixed enum), grouped by theme:

- Markets & Trading: `price-analysis`, `market-news`, `exchange-news`, `derivatives`, `market-manipulation`
- Technology & Development: `protocol-development`, `smart-contracts`, `consensus-mechanisms`, `scaling-solutions`, `interoperability`, `developer-tools`, `open-source`
- Regulation & Legal: `regulatory-news`, `compliance`, `legal-cases`, `government-policy`, `tax-policy`, `enforcement-actions`
- DeFi & Protocols: `defi-protocols`, `lending-borrowing`, `dexes`, `yield-farming`, `staking`, `liquidity`
- NFTs & Digital Collectibles: `nft-markets`, `nft-collections`, `digital-art`, `gaming-nfts`, `nft-utility`, `metaverse`
- Institutional & Enterprise: `institutional-adoption`, `etf-news`, `corporate-treasury`, `custody-solutions`, `traditional-finance`
- Security & Privacy: `security-breaches`, `wallet-security`, `privacy-tech`, `cryptography`, `audit-reports`
- Economics & Tokenomics: `monetary-policy`, `tokenomics`, `mining-news`, `stablecoin-news`, `cbdc`
- Ecosystem & Community: `community-governance`, `ecosystem-growth`, `partnerships`, `funding-news`, `airdrops`
- Education & Analysis: `educational-content`, `research-analysis`, `opinion-editorial`, `interviews`, `case-studies`

**NOTE:** `regulatory-news` and `compliance` are distinct categories - don't conflate them. Category names are matched exactly; an unknown/misspelled category returns `[]` silently rather than erroring, so confirm results are non-empty. Comma-separated multi-category is supported (e.g. `defi-protocols,market-news`).

## Gotchas

- **P0**: Date params must be full ISO 8601 (`2026-05-01T00:00:00Z`).
- Unicode: smart quotes may render as `???` - known hosted issue, cosmetic only.
- No totalCount in response - pagination is offset-based without total.
- Daily summaries start from 2026-04-08. Earlier dates return empty.
- `limit` max is 500 per call. Use `offset` for more.
