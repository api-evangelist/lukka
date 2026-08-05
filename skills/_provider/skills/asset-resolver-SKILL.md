---
name: asset-resolver
description: >
  Resolve customer-provided white paper codes (ticker symbols like BTC, ETH, GALA) to Lukka asset codes
  and retrieve Lukka Prime prices. Use when: (1) customer provides white paper codes / ticker symbols and
  needs prices, (2) mapping white paper codes to Lukka asset codes, (3) identifying the correct asset among
  duplicates, (4) bulk asset resolution for client deliverables, (5) user says "resolve" + asset/ticker/code.
argument-hint: "<ticker codes, e.g. BTC,ETH,GALA,SOL>"
---

# Asset Resolver

**STANDALONE**: This skill provides the white paper code to Lukka canonical code mapping (BTC->XBT, SOL->SOLN, GALA->GALAT3) and resolution workflow logic. **SUPERCHARGED**: When Lukka-RefData and Lukka-Pricing MCP servers are connected, it resolves codes live with exchange mapping counts, status validation, and real-time Lukka Prime prices.

Resolve white paper codes to Lukka asset codes and fetch Lukka Prime prices using the hosted Lukka-RefData and Lukka-Pricing MCP servers.

Resolution path on this surface: alias map -> get_asset_mappings(entityAssetCode) -> get_asset_by_id.

## Critical: the input is a STREET TICKER, not a `lukkaAssetCode`

User input (`TKN`, `GALA`, `BTC`) is a **street ticker**. `lukkaAssetCode` is Lukka's **internal canonical code** - these are NOT the same. On ticker collisions Lukka keeps the bare code on the **oldest** asset and **suffixes** every newer one. Example - a collided ticker `TKN`:

| lukkaAssetCode | Asset | Status |
|---|---|---|
| `TKN` | older asset holding the bare code | **Inactive / Contract Migrated** |
| `TKN01` (group primary) | current asset (multi-chain) | Active |
| `TKNA1C`, `TKN289`, ... | current asset (per chain) | Active |

So `get_asset_mappings({ lukkaAssetCode: "TKN" })` silently returns the **older inactive asset** (status Inactive / Contract Migrated, no price) - never the current one. **Never resolve a raw ticker through `lukkaAssetCode`.** Resolve through `entityAssetCode` (the street-ticker side of the mapping).

> `whitepaperCode` / `commonStreetCode` are NOT filterable on `get_asset_mappings` in this hosted environment - they live only in `get_asset_by_id` -> `nameDetails[]` and are used to **confirm** a candidate after you have its LID, not to search.

## Workflow: Resolve Street Ticker to Lukka Prime Price

### Step 1: Translate known aliases (fast path for majors)

A handful of majors have a clean, unique canonical code. If the input is in this map, the alias IS a unique `lukkaAssetCode` and you may skip straight to Step 3 with it:

| Street ticker | Lukka code | Notes |
|-------------|------------|-------|
| BTC | XBT | ISO 4217 |
| SOL | SOLN | NOT "SOL" - a name search on "SOL" returns unrelated ERC-20 tokens sharing the ticker |
| GALA | GALAT3 | Contract migrated from GALAG |
| MATIC | MATIC (watch for POL) | Polygon migration possible |
| ETH, ADA, DOGE, USDT, USDC | same | Straightforward |

**Canonical Lukka Asset IDs (LIDs) for majors**: XBT=LA6EV2NKQ95, ETH=LA8VRAGEMT1, SOLN=LA1DB6PX4M7.

If the input is **not** in this map, treat it as a raw street ticker and go to Step 2.

### Step 2: Find candidates by street ticker

```
mcp__Lukka-RefData__refdata___get_asset_mappings({
  entityAssetCode: "{ticker}",
  mappingStatus: "Active",
  detailed: true,
  limit: 200
})
```

`entityAssetCode` is the exchange/blockchain-side ticker - this is the only filterable street-ticker field. Collect the **distinct `lukkaAssetId` values** from the rows (one real asset spans many mapping rows - e.g. a multi-chain fund can have 10+ chain mappings).

(For an aliased major from Step 1, you may instead query `lukkaAssetCode: "{alias}"` directly - the alias is unique so no collision risk.)

### Step 3: Validate each distinct candidate + collapse to asset groups

For each distinct `lukkaAssetId`:

```
mcp__Lukka-RefData__refdata___get_asset_by_id({ assetLid: "{lid}" })
```

Apply, in order:
1. **Drop inactive assets**: reject `statusDetails[].assetStatus == "Inactive"` or `assetSubstatus == "Contract Migrated"` (this removes the older inactive holder of the bare ticker). `mappingStatus: "Active"` does NOT do this - the mapping is active even though the asset is inactive.
2. **Collapse by group**: if `assetGroupDetails[].assetGroupList` is present, the candidate is one chain-variant of a group. Replace it with the group's **`isPrimaryAsset: true`** LID. A multi-chain asset's chain-variants all collapse to one group primary LID.
3. **Dedupe** to distinct active asset groups (+ any standalone active assets).

### Step 4: Resolve or disambiguate

- **Exactly one** active group/asset survives -> resolve to it (its group primary). Continue to pricing.
- **More than one** survives -> the ticker is genuinely ambiguous (e.g. a large institutional fund vs a same-ticker token on another chain vs an unrelated token sharing the ticker). **Do not auto-pick.** Present all surviving candidates (name, blockchain, group, quality score if available) and prompt the user to choose. In bulk mode (`/resolve-assets`) do not block the batch - emit an `AMBIGUOUS` row listing the candidates and move on.

If the chosen/sole asset is itself `Contract Migrated`, follow `successorDetails[].successorLukkaId`.

### Step 5: Fetch price

```
mcp__Lukka-Pricing__pricing___get_latest_prices({
  sourceId: 1000,
  pairCodes: "{lukkaAssetCode}-USD"
})
```

Fallback to `sourceId: 2000` (Intraday) if EOD (sourceId 1000) returns empty.

## Ticker Collision Resolution

When the same street ticker maps to multiple Lukka assets (one migrated/inactive, one or more active):

1. Search by `entityAssetCode: "{ticker}"` (NOT `lukkaAssetCode` - the bare code belongs to the oldest asset, often inactive)
2. Reject `Inactive` / `Contract Migrated` assets at the `get_asset_by_id` level
3. Collapse chain-variants to the group's `isPrimaryAsset: true` LID
4. One active group -> resolve it. More than one -> prompt for disambiguation (never auto-pick)
5. Follow `successorDetails` if the resolved asset is itself migrated

## Anti-Patterns

1. **Do NOT** resolve a raw street ticker via `lukkaAssetCode` - on collisions the bare code is the oldest (often inactive) asset (e.g. `lukkaAssetCode:"TKN"` returns the older migrated asset, never the current one). Use `entityAssetCode`.
2. **Do NOT** call `list_assets({ assetCode: ... })` - `list_assets` filters by `lukkaAssetIds` / `assetGroupId`; code resolution goes through `get_asset_mappings`.
3. **Do NOT** auto-pick the highest-mapping candidate on a collided ticker - a newer unrelated token can out-rank the intended asset. Prompt instead.
4. **Do NOT** trust `mappingStatus:"Active"` to filter out inactive assets - the mapping is active even when the asset is `Contract Migrated`. Validate at the asset level.
5. **Do NOT** hardcode asset codes - tokens migrate, always verify via workflow.
6. **Do NOT** pass intervals in seconds - always milliseconds (`86400000`, not `86400`).
7. **Name lookups** resolve through RefData mappings: alias map + `entityAssetCode` lookup.
8. **Do NOT** filter `get_asset_mappings` by `whitepaperCode`/`commonStreetCode` - not supported here; they only confirm a candidate via `get_asset_by_id` -> `nameDetails[]`.

## Output Format

| White Paper Code | Lukka Asset Code | Asset LID | Active Mappings | Price (USD) | Confidence |
|---|---|---|---|---|---|
| BTC | XBT | LA6EV2NKQ95 | 103 | 70,129.00 | HIGH |
| GALA | GALAT3 | LA8OQ23Y4U4 | 103 | 0.00349 | HIGH |
| SOL | SOLN | LA1DB6PX4M7 | 295 | 86.48 | HIGH |

**Confidence levels**:
- **HIGH**: 10+ active exchange mappings + price available
- **MEDIUM**: 3-9 active mappings OR price available but few mappings
- **LOW**: 1-2 active mappings - flag for human review
- **UNRESOLVED**: 0 mappings or no price - escalate

## Examples

**User**: "Resolve GALA"

1. Alias map: `GALA` -> candidate `GALAT3`
2. `get_asset_mappings({ lukkaAssetCode: "GALAT3", mappingStatus: "Active" })` - confirm active, pull `lukkaAssetId`
3. `get_asset_mappings({ lukkaAssetCode: "GALAT3", mappingStatus: "Active" })` - 103 mappings
4. `get_latest_prices({ sourceId: 1000, pairCodes: "GALAT3-USD" })` - $0.00336 (sourceId 1000 = Lukka Prime EOD)
5. Result: GALA -> GALAT3, $0.00336, HIGH confidence

**User**: "Client sent: BTC, ETH, SOL, GALA - need Lukka Prime EOD prices"

1. Translate via alias map: BTC->XBT, ETH->ETH, SOL->SOLN, GALA->GALAT3
2. Parallel: `list_assets` + `get_asset_mappings` for each canonical code
3. Parallel: `get_latest_prices` for XBT-USD, ETH-USD, SOLN-USD, GALAT3-USD
4. Present table with confidence indicators
