---
description: Bulk resolve white paper codes to Lukka asset codes and prices
argument-hint: "<codes, e.g. BTC,ETH,GALA,MATIC>"
allowed-tools:
  - mcp__Lukka-Pricing__pricing___get_latest_prices
  - mcp__Lukka-RefData__refdata___get_asset_by_id
  - mcp__Lukka-RefData__refdata___get_asset_mappings
---

# /resolve-assets

> See [CONNECTORS.md](../CONNECTORS.md) for tool details.

## Usage

/resolve-assets $ARGUMENTS

Parse $ARGUMENTS as comma-separated white paper codes.

## Flow

Uses the **asset-resolver** skill workflow. Resolution path: alias map -> `get_asset_mappings(entityAssetCode)` -> `get_asset_by_id`. The input is a **street ticker**, not a `lukkaAssetCode` - resolving a raw ticker through `lukkaAssetCode` returns the oldest (often inactive) holder of a collided code (the older migrated asset, never the current one). Aliases (BTC -> XBT, SOL -> SOLN, GALA -> GALAT3) are unique canonical codes and resolve directly.

1. **Per ticker**:
   - In alias map -> use the unique canonical code: `get_asset_mappings({ lukkaAssetCode: "{alias}", mappingStatus: "Active", limit: 200 })`.
   - Not in alias map -> raw ticker: `mcp__Lukka-RefData__refdata___get_asset_mappings({ entityAssetCode: "{ticker}", mappingStatus: "Active", detailed: true, limit: 200 })` [~~refdata]. Collect distinct `lukkaAssetId` values.
2. **Validate + collapse**: `mcp__Lukka-RefData__refdata___get_asset_by_id({ assetLid })` [~~refdata] per distinct LID -> drop `Inactive`/`Contract Migrated`, collapse chain-variants to the group `isPrimaryAsset: true` LID, dedupe to active groups.
3. **Resolve or flag**: one active group -> winner. More than one -> emit `AMBIGUOUS` (see below); do **not** block the batch or auto-pick.
4. **Price**: `mcp__Lukka-Pricing__pricing___get_latest_prices({ sourceId: 1000, pairCodes: "{lukkaAssetCode}-USD" })` [~~pricing] (sourceId 1000 = Lukka Prime EOD)

## Output

### Asset Resolution Results

| White Paper Code | Lukka Code | Lukka ID | Active Mappings | Price (USD) | Confidence |
|-----------------|------------|----------|-----------------|-------------|------------|
| BTC | XBT | LA6EV2NKQ95 | 52 | $XX,XXX | HIGH |
| (collided ticker) | (multiple) | - | - | - | AMBIGUOUS |

Note: Lukka IDs always follow the format `LA` + 9 alphanumeric characters (e.g. `LA6EV2NKQ95`). Never display abbreviated or placeholder formats like `lid-xxx`.

**Confidence levels**: HIGH (10+ active mappings), MEDIUM (3-9), LOW (1-2), UNRESOLVED (0), AMBIGUOUS (>1 active asset group shares the ticker)
- If result count equals limit (200), display as "200+ (truncated)" and confidence as "HIGH (truncated)"
- For `AMBIGUOUS` rows, follow the table with a short list of competing candidates (name, blockchain, group primary LID) so the user can re-run `/asset-report` on the right one. Never silently pick one.
