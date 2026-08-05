---
description: Ping each hosted MCP server with one cheap tool, report auth, dispatch, and known drift.
argument-hint: ""
allowed-tools:
  - mcp__Lukka-AML__AML___generateAmlScoringReport
  - mcp__Lukka-UDA__UDA___Get_newest_block_blockheight_v2
  - mcp__Lukka-Pricing__pricing___get_latest_index_prices
  - mcp__Lukka-Pricing__pricing___list_pricing_sources
  - mcp__Lukka-RefData__refdata___get_asset_mappings
  - mcp__Lukka-RefData__refdata___get_entity_by_code
  - mcp__Lukka-Analytics__analytics___getImpliedRates
  - mcp__Lukka-News__news___getDailySummaries
  - mcp__Lukka-Predmar__prediction-markets___GetExchanges
---

# /health

> See [CONNECTORS.md](../CONNECTORS.md) for server details.

## Usage

/health

## Purpose

Smoke-test the seven hosted MCP servers and flag known drift conditions in under 10 seconds. Use this:
- Right after `claude plugin install lukka` + OAuth authentication, to confirm every server is reachable
- Before starting a long session, to catch availability changes early
- When a downstream command fails mysteriously, to isolate server-level issues from flow-level issues

## Flow (parallel where possible)

### Phase A - Server reachability (7 smoke calls, parallel)

Run the cheapest dispatchable tool per server. Every call should return a shaped payload without ValidationException.

1. `mcp__Lukka-Pricing__pricing___list_pricing_sources({})` [~~pricing]
2. `mcp__Lukka-RefData__refdata___get_asset_mappings({ lukkaAssetCode: "XBT", mappingStatus: "Active", limit: 1 })` [~~refdata]
3. `mcp__Lukka-AML__AML___generateAmlScoringReport({ address: "0x13203aB9C9f054fe8B4671bC0EEC3ab26D2f3477", address_type: "ETH" })` [~~aml]
4. `mcp__Lukka-UDA__UDA___Get_newest_block_blockheight_v2({ blockchain: "eth" })` [~~onchain]
5. `mcp__Lukka-Predmar__prediction-markets___GetExchanges({ limit: "1", offset: "0", primary_location: "", vasp_classification: "", vast_type: "" })` [~~predmar]
6. `mcp__Lukka-News__news___getDailySummaries({ limit: 1 })` [~~news]
7. `mcp__Lukka-Analytics__analytics___getImpliedRates({ pairCode: "XBT-USD", lukkaEntityCode: "LUKKA", curveType: "INST_FWD", modelType: "NSS", source: "FUTURES", rateType: "BASIS", timeRange: "1D", timeTo: "{now}", limit: 1 })` [~~analytics] - note `timeRange` MUST be anchored with `timeTo` (or `timeFrom`); an un-anchored `timeRange` returns HTTP 400 by design.

Record per server: dispatched (yes/no), response shape ok (yes/no), latency.

### Phase B - Known-drift checks (3 targeted calls, parallel)

8. `mcp__Lukka-RefData__refdata___get_entity_by_code({ code: "COINBASE" })` [~~refdata] - `COINBASE` is NOT a canonical code and MUST NOT resolve to an Exchange record; the canonical Coinbase code is `CPRO`. Assert the result is not type `Exchange` (routing correctness check). If it now returns a canonical exchange record, flag as a data change and prompt the user to re-validate canonical codes.
9. `mcp__Lukka-RefData__refdata___get_entity_by_code({ code: "CPRO" })` [~~refdata] - MUST return Coinbase Pro (type `Exchange`, jurisdiction `USA`, amlKycProcess `Y`).
10. `mcp__Lukka-Pricing__pricing___get_latest_index_prices({ sourceId: 3000, pairCodes: "XBT-USD" })` [~~pricing] - index-price availability probe (sourceId 3000 = Hourly; Reference Rate = 5001/5002, requires access). Two acceptable outcomes: (a) payload -> access enabled; (b) 404 -> use EOD/Intraday path. Any other error (401/403/5xx) is a drift signal.

### Phase C - Processor sanity (local, no MCP)

11. Run `process_aml_report.py` against the bundled synthetic fixture `${CLAUDE_PLUGIN_ROOT}/tests/fixtures/aml/scoring_sample.raw` and confirm output > 100 bytes and contains `cscore`. This verifies the processor works without depending on external state.

## Output

```
Lukka Plugin Health - {date} {time}

Server availability (Phase A)
| # | Server | Tool | Dispatched | Shape OK | Notes |
|---|--------|------|-----------|----------|-------|
| 1 | Lukka-Pricing | list_pricing_sources | YES / NO | YES / NO | {count} active sources |
| 2 | Lukka-RefData | get_asset_mappings(XBT) | YES / NO | YES / NO | lukkaAssetId = {lid} |
| 3 | Lukka-AML | generateAmlScoringReport | YES / NO | YES / NO | score {n}, level {lvl} |
| 4 | Lukka-UDA | Get_newest_block_blockheight_v2(eth) | YES / NO | YES / NO | block {n} |
| 5 | Lukka-Predmar | GetExchanges | YES / NO | YES / NO | {count} venues |
| 6 | Lukka-News | getDailySummaries(limit:1) | YES / NO | YES / NO | date: {date}, articles: {count} |
| 7 | Lukka-Analytics | getImpliedRates(XBT-USD) | YES / NO | YES / NO | {n} tenors, businessTime {ts} |

Known drift (Phase B)
| Check | Expected | Got | Status |
|-------|----------|-----|--------|
| COINBASE non-canonical | type != Exchange (use CPRO) | {actual} | PASS / FAIL |
| CPRO canonical | type=Exchange, Coinbase Pro | {actual} | PASS / FAIL |
| get_latest_index_prices(3000) | price payload (access enabled) OR 404 (use EOD/Intraday path) | {actual} | AVAILABLE / EOD-PATH / DRIFT |

Summary
- Overall: {PASS / DEGRADED / FAIL}
- Actions: {list}
```

## Interpretation

- **PASS**: all 7 servers up, all drift checks match documented expectations. Safe to run any command.
- **DEGRADED**: one server down, or one drift check flipped. Command-level impact listed in Actions.
- **FAIL**: two or more servers down, or both entity-code checks broken. Plugin is not usable; check `/mcp` auth.
- **All servers unauthenticated / no credentials**: if every server fails to dispatch, you likely have not authenticated (or have no Lukka account). Authenticate each server via `/mcp`. No account or keys yet? Request access at **https://lukka.tech/plugin**. For setup or access problems, contact **plugin-support@lukka.global**.

## Common follow-ups

- Missing `address_type` on AML: confirm you are calling `generateAmlScoringReport` with `address_type` (UPPERCASE chain, e.g. `ETH`), not `blockchain`.
- `GetExchanges` ValidationException: confirm all 5 required params passed.
- `get_asset_mappings` returns empty for XBT: data may not be available for that code; retry with `lukkaAssetCode: "ETH"`.
- `Get_newest_block_blockheight_v2` 5xx: try `Get_newest_block_blockheight` (v1 fallback).
