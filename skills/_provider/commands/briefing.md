---
description: Daily macro market overview with top assets, prices, volume, and sentiment
argument-hint: ""
allowed-tools:
  - mcp__Lukka-Pricing__pricing___get_candles
  - mcp__Lukka-Pricing__pricing___get_latest_market_caps
  - mcp__Lukka-Pricing__pricing___get_latest_prices
  - mcp__Lukka-News__news___getDailySummaries
  - mcp__Lukka-Predmar__prediction-markets___GetMarketRecap
  - mcp__Lukka-Predmar__prediction-markets___GetPredictionMarkets
---

# /briefing

> See [CONNECTORS.md](../CONNECTORS.md) for tool details.

## Usage

/briefing

## Flow

0. **Read preferences (optional)**: if `${CLAUDE_PLUGIN_ROOT}/settings/preferences.local.md` exists, read `Top N` and use it as the market-cap limit in steps 1-2 below. Rules:
   - No file, or key absent -> default **20**.
   - Non-numeric / unparseable value (e.g. `abc`) -> fall back to **20** (do not error).
   - Value above **50** -> clamp to **50** (the documented max) and note the clamp to the user.
   - `Top N` is the only preference key currently consumed; other keys in the example file are placeholders (not yet wired).

1. **Top N by market cap (current)** (always pass `order:"DESC"` explicitly):
   `mcp__Lukka-Pricing__pricing___get_latest_market_caps({ limit: {TopN}, order: "DESC" })` [~~pricing]
2. **Top N by market cap (24h ago)** - for cap delta:
   `mcp__Lukka-Pricing__pricing___get_latest_market_caps({ limit: {TopN}, order: "DESC", asOf: "{24h_ago_ISO}" })` [~~pricing]
3. **Current prices** (realtime): `mcp__Lukka-Pricing__pricing___get_latest_prices({ sourceId: 2000, pairCodes: top20_pairs })` [~~pricing] (sourceId 2000 = Lukka Prime Intraday)
4. **24h candles for top 5**: Per asset, `mcp__Lukka-Pricing__pricing___get_candles({ sourceId: 4002, pairCode, interval: 86400000, from: 24h_ago, to: now })` [~~pricing] (sourceId 4002 = Coinbase Pro, schema default)
   - Extract `openPrice`/`closePrice` for 24h change: `(close - open) / open * 100`
   - Extract `volume` for 24h volume
5. **Prediction market sentiment**: `mcp__Lukka-Predmar__prediction-markets___GetPredictionMarkets({ search: "bitcoin", sort_by: "total_volume_usd_1hr", sort_dir: "desc", limit: "5" })` [~~predmar]
6. **Market recap**: `mcp__Lukka-Predmar__prediction-markets___GetMarketRecap({})` [~~predmar]
7. **News Digest** (today's or most recent): `mcp__Lukka-News__news___getDailySummaries({ limit: 1 })` [~~news]
   - If today is Monday, summary covers Saturday + Sunday (weekend digest).
   - Summaries available from 2026-04-08.

> Implied rates and volatility surface are available via `Lukka-Analytics` server. Optionally include if user requests rate/vol context.

## Output

### Market Briefing - {date}

**Top 5 by Market Cap** (full analytics)

| # | Asset | Price (USD) | 24h Change | 24h Volume | Market Cap | Cap Change 24h |
|---|-------|------------|------------|------------|------------|----------------|
| 1 | XBT | $XX,XXX | +X.X% | $X.XB | $X.XT | +X.X% |

- 24h Change = from candles `(closePrice - openPrice) / openPrice * 100`
- 24h Volume = from candles `volume` field
- Cap Change 24h = `(current_cap - cap_24h_ago) / cap_24h_ago * 100` (from step 1 vs step 2)

**Rank 6-20** (snapshot only - no candle data fetched for these)

| # | Asset | Price (USD) | Market Cap |
|---|-------|------------|------------|
| 6 | ... | $X,XXX | $X.XB |

**24h Highlights** (derived from top 5 candles only)
- Top gainer: {asset} (+X.X%)
- Top loser: {asset} (-X.X%)
- Volume leader: {asset} ($X.XB)

**Market Sentiment** (Prediction Markets)
- {event}: {probability}%
  - Probability = `yes.latest_price * 100` for binary markets

**News Digest**
- Overall sentiment: {overallSentiment}
- Articles today: {articleCount}
- Top headlines: {notableEvents[0..2] titles}
- Key themes: {keyThemes[0..2]}

## Pagination

`get_latest_market_caps` returns up to `limit` rows per call. If you need beyond top 100, use cursor-based pagination: pass `cursor` from previous response's `nextCursor` field. For this command, limit:20 is sufficient.

## Scheduling

Ideal as a daily command at 8:00 AM. Re-run manually, or drive it on a recurring schedule with whatever task-scheduling mechanism your Claude Code setup provides.
