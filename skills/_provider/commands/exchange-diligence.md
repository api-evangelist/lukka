---
description: Exchange compliance profile with jurisdiction, VASP status, and services
argument-hint: "<exchange code, e.g. CPRO, KRAK, BINA>"
allowed-tools:
  - mcp__Lukka-RefData__refdata___get_asset_by_id
  - mcp__Lukka-RefData__refdata___get_asset_mappings
  - mcp__Lukka-RefData__refdata___get_entity_by_code
  - mcp__Lukka-RefData__refdata___list_vasps
---

# /exchange-diligence

> See [CONNECTORS.md](../CONNECTORS.md) for tool details.

## Usage

/exchange-diligence $ARGUMENTS

### Canonical codes (IMPORTANT)

Do **not** pass free-form names. `COINBASE` is not a canonical code and does not resolve to the exchange. Use canonical Lukka codes:

| Venue | Code |
|-------|------|
| Coinbase Pro | `CPRO` |
| Coinbase (parent) | `CBSE` |
| Coinbase Prime | `CPRM` |
| Kraken | `KRAK` |
| Binance | `BINA` |
| Binance US | `BIUS` |
| Gemini | `GMNI` |
| Bitstamp | `BSTP` |
| OKX | `OKX` |

If the user types a free-form name, translate it to the code above before calling.

## Flow

> **NOTE:** This command builds the exchange profile from entity + VASP + asset-mapping data.

1. **Entity info**: `mcp__Lukka-RefData__refdata___get_entity_by_code({ code: "$ARGUMENTS" })` [~~refdata]
2. **VASP data**: `mcp__Lukka-RefData__refdata___list_vasps({ limit: 100 })` [~~refdata] -> find the row where `lukkaEntityCode` == `$ARGUMENTS`. VASPs are NOT filterable by `entityCode` (it only bridges to mappings, not VASPs) - filter client-side on `lukkaEntityCode`; paginate via `offset` if not on the first page.
3. **Asset listings + services**: `mcp__Lukka-RefData__refdata___get_asset_mappings({ entityCode: "$ARGUMENTS", mappingStatus: "Active", limit: 100 })` [~~refdata]
   - Count active listings
   - Aggregate `supportedFunctionList` across all mappings: for each flag (`tradingSupported`, `custodySupported`, `lendingSupported`, `savingsSupported`, `stakingSupported`), set to "yes" if *any* mapping has it `true`

### Optional: `--flag-risky` mode (expensive - user must explicitly request)

4. **Flag risky assets**: iterate mappings from step 3 and per row call `mcp__Lukka-RefData__refdata___get_asset_by_id({ assetLid: "{lukkaAssetId}" })` [~~refdata] to inspect `assetDetails.potentiallySuspicious` / `potentiallyMemeCoin` flags.
   - Do NOT run by default (100+ calls for large exchanges)

## Output

### Exchange Due Diligence: {exchange_name}

**Entity Profile**
- Code: {entityCode}
- Type: {entityType}
- Country: {country}
- Status: {status}

**Regulatory**
- Jurisdiction: {jurisdiction} (primary; subsidiaries not enumerated)
- AML/KYC Process: {amlKycProcess}
- VASP Registration: {status from list_vasps} or "N/A" if no VASP registration found
- Note: Regulatory registrations per jurisdiction are provided via `list_vasps` (match `lukkaEntityCode` client-side).

**Services** (aggregated from `supportedFunctionList` across all active mappings)
- Spot trading: {yes/no}
- Derivatives: {yes/no}
- Lending: {yes/no}
- Staking: {yes/no}
- Custody: {yes/no}
- Savings: {yes/no}
- Note: based on supported asset function flags aggregated across listings

**Asset Coverage**
- Active listings: {count} (from step 3 `get_asset_mappings`)

> Suspicious/meme coin counts are not resolved by default (requires N individual asset lookups). Use `--flag-risky` to iterate all mappings and check `potentiallySuspicious` / `potentiallyMemeCoin` flags.

**Risk Flags**
- {flag descriptions from entity/VASP data}
