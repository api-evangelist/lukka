---
description: Activity-focused asset report with trends, volume, and recent events
argument-hint: "<asset, e.g. UNI or Uniswap>"
allowed-tools:
  - mcp__Lukka-Pricing__pricing___get_candles
  - mcp__Lukka-Pricing__pricing___get_latest_market_caps
  - mcp__Lukka-Pricing__pricing___get_market_capitalization
  - mcp__Lukka-Pricing__pricing___get_market_depth
  - mcp__Lukka-Pricing__pricing___get_vwap
  - mcp__Lukka-RefData__refdata___get_asset_by_id
  - mcp__Lukka-RefData__refdata___get_asset_crypto_actions
  - mcp__Lukka-RefData__refdata___get_asset_mappings
---

# /asset-activity

> See [CONNECTORS.md](../CONNECTORS.md) for tool details.

## Usage

/asset-activity $ARGUMENTS

## Flow

1. **Resolve code -> assetLid**: `mcp__Lukka-RefData__refdata___get_asset_mappings({ lukkaAssetCode: "$ARGUMENTS", mappingStatus: "Active", limit: 5 })` [~~refdata] -> take `lukkaAssetId` from first row
   - Name lookups resolve through RefData mappings; translate free-form names to canonical codes first; `list_assets` filters by `lukkaAssetIds` / `assetGroupId`, code resolution goes through `get_asset_mappings`
2. **Metadata**: `mcp__Lukka-RefData__refdata___get_asset_by_id({ assetLid })` [~~refdata]
3. **Price trend**: `mcp__Lukka-Pricing__pricing___get_candles({ sourceId: 4002, pairCode: "{code}-USD", interval: 86400000, from: 30d_ago, to: now })` [~~pricing] (sourceId 4002 = Coinbase Pro, schema default)
   - 30d change % = `(last_close - first_open) / first_open * 100`
4. **VWAP (7d)**: `mcp__Lukka-Pricing__pricing___get_vwap({ pairCode: "{code}-USD", sourceIds: top_exchange_ids, from: 7d_ago, to: now })` [~~pricing]
   - max 7-day range per call
5. **Market cap + rank**:
   - `mcp__Lukka-Pricing__pricing___get_market_capitalization({ pairCode: "{code}-USD", limit: 1 })` [~~pricing] -> USD cap. Takes single `pairCode` (BASE-USD), NOT `assetCodes`.
   - `mcp__Lukka-Pricing__pricing___get_latest_market_caps({ limit: 100, order: "DESC" })` [~~pricing] -> find position of `{code}` in list; if not present, rank is "100+"
6. **Liquidity** (deterministic exchange selection: top 5 from `get_pair_mappings` where exchange is in {KRAK, CPRO, BINA, BFNX, GMNI}):
   Per exchange: `mcp__Lukka-Pricing__pricing___get_market_depth({ sourceIds: "{single}", baseAssetCodes: "{code}", counterAssetCodes: "USD" })` [~~pricing]
7. **Exchange coverage**: `mcp__Lukka-RefData__refdata___get_asset_mappings({ lukkaAssetCode: "{code}", mappingStatus: "Active" })` [~~refdata]
8. **Recent events**: `mcp__Lukka-RefData__refdata___get_asset_crypto_actions({ assetLid: "{LID}", domain: "assets" })` [~~refdata] - takes `assetLid` (LA-code), NOT `assetCode`; `domain` required.

## Output

### Activity Report: {asset_name} ({code})

**Price Trend (30d)**
- Current: ${price}
- 30d high: ${high}
- 30d low: ${low}
- 30d change: {pct}%

**Volume (7d)**
- 7d avg daily: ${vol}
- 7d VWAP: ${vwap}
- VWAP deviation: {bps} bps

**Market Cap**: ${cap} (rank #{rank})

**Liquidity Snapshot**
| Exchange | Bid Depth | Ask Depth | Spread |
|----------|-----------|-----------|--------|

**Exchange Coverage**: {count} active listings

**Recent Events**
| Date | Event | Description |
|------|-------|-------------|
