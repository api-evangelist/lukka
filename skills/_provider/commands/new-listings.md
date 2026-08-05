---
description: Monitor new exchange listings since a given date
argument-hint: "<exchange code> [since:YYYY-MM-DD]"
allowed-tools:
  - mcp__Lukka-Pricing__pricing___get_latest_prices
  - mcp__Lukka-RefData__refdata___get_asset_by_id
  - mcp__Lukka-RefData__refdata___get_asset_mappings
---

# /new-listings

> See [CONNECTORS.md](../CONNECTORS.md) for tool details.

## Usage

/new-listings $ARGUMENTS

Parse $ARGUMENTS for exchange code (use canonical Lukka codes: `CPRO`, `KRAK`, `BINA`, `GMNI`, etc. - never `COINBASE`) and optional `since` date (default: 7 days ago).

## Flow

1. **Recent mappings**: `mcp__Lukka-RefData__refdata___get_asset_mappings({ entityCode: "{code}", mapProcessedStartTime: "{since}T00:00:00Z", mappingStatus: "Active", detailed: true, limit: 100 })` [~~refdata]
2. **Per new listing**: take `lukkaAssetId` from the mapping row, then `mcp__Lukka-RefData__refdata___get_asset_by_id({ assetLid: "{lukkaAssetId}" })` [~~refdata] for metadata (`list_assets` filters by `lukkaAssetIds` / `assetGroupId`; code resolution goes through `get_asset_mappings`)
3. **Prices**: `mcp__Lukka-Pricing__pricing___get_latest_prices({ sourceId: 2000, pairCodes: "{code}-USD" })` [~~pricing] for each (sourceId 2000 = Lukka Prime Intraday)
   - If price returns empty/404 for a fresh listing, display "not yet indexed" instead of blank

## Output

### New Listings on {exchange} since {date}

Found **{count}** new listings.

| # | Asset Code | Asset Name | Blockchain | First Active | Price (USD) |
|---|-----------|------------|------------|--------------|-------------|

- First Active = `mapEffectiveStartTime` (when trading actually began); falls back to `mapProcessedStartTime` (when Lukka recorded) if effective is null
- Source shown in parentheses: "(effective)" or "(processed)"

**Summary**: {count} new assets listed. {highlights}.
