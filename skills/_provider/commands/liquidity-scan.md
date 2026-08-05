---
description: Cross-exchange liquidity analysis with depth, spread, and VWAP
argument-hint: "<pair, e.g. SOLN-USD>"
allowed-tools:
  - mcp__Lukka-Pricing__pricing___get_candles
  - mcp__Lukka-Pricing__pricing___get_market_depth
  - mcp__Lukka-Pricing__pricing___get_vwap
  - mcp__Lukka-Pricing__pricing___list_pricing_sources
  - mcp__Lukka-RefData__refdata___get_pair_mappings
---

# /liquidity-scan

> See [CONNECTORS.md](../CONNECTORS.md) for tool details.

## Usage

/liquidity-scan $ARGUMENTS

## Flow

1. **Find exchanges for pair**: `mcp__Lukka-RefData__refdata___get_pair_mappings({ lukkaPairCode: "{pair}", mappingStatus: "Active", detailed: true, limit: 1000 })` [~~refdata] -> list of entityCodes trading this pair
   - **Page fully** (a flagship pair has 80+ venues). If `next` is non-null, keep paging - a truncated page silently drops exchanges (e.g. wrongly concluding BINA does not trade XBT-USD when it does). Never treat an exchange as absent from a partial page.
2. **Map to sourceIds**: `mcp__Lukka-Pricing__pricing___list_pricing_sources({})` [~~pricing] -> filter ACTIVE sources; match entityCodes to sourceIds
3. **Parallel per exchange** (via `market-data-gatherer` agent) - one call per sourceId:
   - `mcp__Lukka-Pricing__pricing___get_market_depth({ sourceIds: "{single_sourceId}", baseAssetCodes: "{base}", counterAssetCodes: "USD" })` [~~pricing]
     **CRITICAL**: base/counter filters mandatory (omitting returns 6.8 MB exchange dump). `sourceIds` accepts comma-separated values (one record per source) - you may batch several exchanges into one `get_market_depth` call to cut round-trips
   - `mcp__Lukka-Pricing__pricing___get_vwap({ pairCode: "{pair}", sourceIds: "{single_sourceId}", from: 24h_ago, to: now })` [~~pricing]
   - `mcp__Lukka-Pricing__pricing___get_candles({ pairCode: "{pair}", sourceId: "{single_sourceId}", interval: 86400000 })` [~~pricing]
4. **Aggregate and rank** by: total depth (bids_1 + asks_1 within 1% of mid), spread, volume

> **Full exchange depth** (optional - very large responses): use `large-data-processing` skill.
> `cp "${CLAUDE_PLUGIN_ROOT}/skills/large-data-processing/references/process_market_depth.py" /tmp/`
> `python3 /tmp/process_market_depth.py /tmp/depth.json --top 20 --min-depth 50000`

## Output

### Liquidity Scan: {pair}

**Exchange Ranking** (by total depth within 1% of mid)

| # | Exchange | Bid Depth (1%) | Ask Depth (1%) | Bid Depth (5%) | Ask Depth (5%) | Spread (bps) | 24h Volume | VWAP |
|---|----------|---------------|---------------|---------------|---------------|--------------|------------|------|

- Bid/Ask Depth (1%) = `bids_1` / `asks_1` fields (liquidity within 1% of mid price)
- Bid/Ask Depth (5%) = `bids_5` / `asks_5` fields (liquidity within 5% of mid - for larger orders)

**Best Execution**
- Tightest spread: {exchange} ({bps} bps)
- Deepest book (1%): {exchange} (${depth})
- Highest volume: {exchange} (${vol})
- Recommended max order (1% slippage): ${size}
  - Formula: `min(bids_1 * mid_price, asks_1 * mid_price) * 0.8` (20% safety buffer)
- Recommended max order (5% slippage): ${size_5pct}
  - Formula: `min(bids_5 * mid_price, asks_5 * mid_price) * 0.8`
