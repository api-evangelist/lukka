---
description: Comprehensive asset deep dive with classification, scores, and exchange coverage
argument-hint: "<asset name or code, e.g. SOLN or Solana>"
allowed-tools:
  - mcp__Lukka-Pricing__pricing___get_latest_prices
  - mcp__Lukka-Pricing__pricing___get_market_capitalization
  - mcp__Lukka-RefData__refdata___get_asset_by_id
  - mcp__Lukka-RefData__refdata___get_asset_crypto_actions
  - mcp__Lukka-RefData__refdata___get_asset_mappings
  - mcp__Lukka-RefData__refdata___get_core_asset_scores
  - mcp__Lukka-RefData__refdata___get_sectors
---

# /asset-report

> See [CONNECTORS.md](../CONNECTORS.md) for tool details.

## Usage

/asset-report $ARGUMENTS

## Flow

1. **Resolve ticker -> assetLid** (resolution path: alias map -> `get_asset_mappings(entityAssetCode)` -> `get_asset_by_id`; `list_assets` filters by `lukkaAssetIds` / `assetGroupId`). The input is a **street ticker**, not a `lukkaAssetCode` - on collisions the bare code is the oldest (often inactive) asset (e.g. `lukkaAssetCode:"TKN"` returns the older migrated asset, never the current one). Resolve via the **asset-resolver** skill:
   - If input is a name (e.g. "Solana") or a known major, translate via the alias map (`XBT`, `ETH`, `SOLN`, `USDC`, `USDT`, ...) - aliased codes are unique, query `lukkaAssetCode` directly.
   - Otherwise treat as a raw ticker: `mcp__Lukka-RefData__refdata___get_asset_mappings({ entityAssetCode: "$ARGUMENTS", mappingStatus: "Active", detailed: true, limit: 200 })` [~~refdata]
   - Collect distinct `lukkaAssetId` values. For each, `get_asset_by_id` -> drop `Inactive`/`Contract Migrated`, collapse chain-variants to the group `isPrimaryAsset: true` LID.
   - **One** active asset survives -> use its LID as `assetLid`. **More than one** (e.g. a large fund vs a same-ticker token on another chain vs an unrelated same-ticker token) -> list candidates and ask the user to pick; do not auto-resolve.
2. **Full metadata**: `mcp__Lukka-RefData__refdata___get_asset_by_id({ assetLid })` [~~refdata]
   - Includes `assetDetails` (stakeable, minable, stable, wrapped, governanceToken), `blockchainDetails`, `governanceDetails`, `participantDetails`
3. **Classification**: `mcp__Lukka-RefData__refdata___get_sectors({ assetLid })` [~~refdata] - 5-tier LDACS
4. **Quality score**: `mcp__Lukka-RefData__refdata___get_core_asset_scores({ assetCode, fromDate: 30d_ago, toDate: today })` [~~refdata]
   - **REQUIRED**: both `fromDate` + `toDate` (ISO 8601), else returns empty
5. **Exchange coverage**: `mcp__Lukka-RefData__refdata___get_asset_mappings({ entityCode: <optional>, lukkaAssetCode, mappingStatus: "Active", limit: 50 })` [~~refdata]
6. **Crypto Actions** (past + future): `mcp__Lukka-RefData__refdata___get_asset_crypto_actions({ assetLid: "{LID}", domain: "assets" })` [~~refdata]
   - Takes `assetLid` (the LA-code resolved in step 1-2), NOT `assetCode`. `domain` is required (`assets`|`derivatives`|`assetMappings`|`pairMappings`).
   - Sort results by date descending (newest first)
7. **Price + market cap**: `mcp__Lukka-Pricing__pricing___get_latest_prices({ sourceId: 1000, pairCodes: "{code}-USD" })` + `mcp__Lukka-Pricing__pricing___get_market_capitalization({ pairCode: "{code}-USD", limit: 1 })` [~~pricing] (sourceId 1000 = Lukka Prime EOD). NOTE `get_market_capitalization` takes a single `pairCode` (BASE-USD), NOT `assetCodes`; for a top-N multi-asset snapshot use `get_latest_market_caps`.

## Output

### Asset Report: {asset_name} ({lukka_code})

**Identity**
- Lukka ID: {assetLid}
- Lukka Code: {lukkaAssetCode}
- Blockchain: {blockchain}
- Type: {assetType}
- Token Standard: {tokenStandard}

**Classification (LDACS)**
- Super Sector > Macro > Mid > Sub > Category

**Scores**
- LCAS: {score}/100 ({rating})

**Market Data**
- Price: ${price} (Lukka Prime EOD)
- Market Cap: ${marketCap}

**Exchange Coverage** ({count} active mappings)
| Exchange | Mapping Code | Status |
|----------|-------------|--------|

**Governance**
- Open Source: {yes/no/Not disclosed}
- Bug Bounty: {yes/no/Not disclosed}
- Foundation: {yes/no/Not disclosed}
- Project Structure: {centralized/decentralized/hybrid/Not classified}

If `governanceDetails` is empty for the asset, render each field as "Not disclosed" and add note: "Governance metadata not maintained for this asset in Lukka reference data."

**Asset Properties**
- Stakeable: {yes/no} | Minable: {yes/no} | Stable: {yes/no} | Wrapped: {yes/no}
- Governance Token: {yes/no} | Precision: {N} | Layer: {L1/L2}

**Recent & Upcoming Events**
| Date | Action | Description | Future? |
|------|--------|-------------|---------|
