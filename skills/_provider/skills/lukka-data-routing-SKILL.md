---
name: lukka-data-routing
description: >-
  Route digital asset and blockchain data queries to the correct Lukka MCP tool across
  7 hosted servers. Use when user asks about: digital asset prices, OHLCV candles,
  market cap, VWAP, order book depth, trading pairs, exchange sources,
  derivatives, implied rates, volatility surface, OTC FX,
  reference data, asset taxonomy, sector classifications,
  entity/custodian/VASP lookups, asset mappings, AML reports, wallet balances,
  on-chain transactions, block data, digital asset news, daily summaries, sentiment analysis,
  or prediction markets.
argument-hint: "<digital asset data question, e.g. price of ETH, AML check on 0x..., Solana block height>"
---

# Lukka Data Routing

**STANDALONE**: This skill provides routing knowledge and domain conventions that help answer digital asset data questions from general knowledge. **SUPERCHARGED**: When the 7 Lukka MCP servers are connected, this skill routes queries to the exact right tool with correct parameters, avoiding common API pitfalls.

Seven hosted MCP servers. Pick the right one on the first try.

## Tool Naming Convention

Production tools are exposed with the prefix `mcp__<Server>__<group>___<tool>` (triple underscore before the tool name). Always call tools with the full wire name - short names (`get_latest_prices`) are NOT dispatched by the hosted runtime.

| Server | Prefix |
|--------|--------|
| Lukka-Pricing | `mcp__Lukka-Pricing__pricing___*` |
| Lukka-RefData | `mcp__Lukka-RefData__refdata___*` |
| Lukka-AML | `mcp__Lukka-AML__AML___*` |
| Lukka-UDA | `mcp__Lukka-UDA__UDA___*` |
| Lukka-Analytics | `mcp__Lukka-Analytics__analytics___*` |
| Lukka-News | `mcp__Lukka-News__news___*` |
| Lukka-Predmar | `mcp__Lukka-Predmar__prediction-markets___*` |

## Resolution and matching approach

- **Asset/pair/market resolution**: use direct filters on refdata tools (`list_assets`, `get_asset_mappings`, `get_pair_mappings`).
- **Derivative instrument matching**: use `get_derivative_source_details` with client-side string match.

## Route the Query

| User asks about | Server | Tool | Ref |
|---|---|---|---|
| Current price | Lukka-Pricing | `get_latest_prices` | [pricing](references/pricing.md) |
| Price history / charts | Lukka-Pricing | `get_historical_prices` or `get_candles` | [pricing](references/pricing.md) |
| Index / reference rate | Lukka-Pricing | `get_latest_index_prices` (Reference Rate access; standard path: sourceId 1000/2000) | [pricing](references/pricing.md) |
| Market cap / top N | Lukka-Pricing | `get_latest_market_caps` with `order:"DESC"` | [pricing](references/pricing.md) |
| VWAP | Lukka-Pricing | `get_vwap` | [pricing](references/pricing.md) |
| Order book / depth | Lukka-Pricing | `get_market_depth` | [pricing](references/pricing.md) |
| Order book snapshot | Lukka-Pricing | `get_order_book_snapshot` | [pricing](references/pricing.md) |
| Spot trades | Lukka-Pricing | `get_spot_trades` | [pricing](references/pricing.md) |
| Exchange sources | Lukka-Pricing | `list_pricing_sources` | [pricing](references/pricing.md) |
| Derivatives trades | Lukka-Pricing | `get_latest_derivative_trades` | [pricing](references/pricing.md) |
| Derivative source detail | Lukka-Pricing | `get_derivative_source_details` | [pricing](references/pricing.md) |
| What is asset X | Lukka-RefData | `get_asset_mappings(lukkaAssetCode)` -> `get_asset_by_id(assetLid)` | [refdata](references/refdata.md) |
| Find assets by trait | Lukka-RefData | `list_assets` + client-side filter | [refdata](references/refdata.md) |
| Exchange mappings | Lukka-RefData | `get_asset_mappings` | [refdata](references/refdata.md) |
| Sectors / taxonomy | Lukka-RefData | `get_sectors` | [refdata](references/refdata.md) |
| LCAS quality scores | Lukka-RefData | `get_core_asset_scores` | [refdata](references/refdata.md) |
| Entities / exchanges | Lukka-RefData | `get_entity_by_code` / `list_entities` | [refdata](references/refdata.md) |
| VASPs | Lukka-RefData | `list_vasps` | [refdata](references/refdata.md) |
| Crypto Actions | Lukka-RefData | `get_asset_crypto_actions` / `get_crypto_actions` | [refdata](references/refdata.md) |
| Pair mapping lookup | Lukka-RefData | `get_pair_mappings` | [refdata](references/refdata.md) |
| AML light check | Lukka-AML | `generateAmlScoringReport` | [aml](references/aml.md) |
| AML full report | Lukka-AML | `generateAmlStandardReport` (pipe through processor) | [aml](references/aml.md) |
| AML deepest report | Lukka-AML | `generateAmlEnhancedReport` (pipe through processor) | [aml](references/aml.md) |
| Wallet balance | Lukka-UDA | `get_address_balance` | [uda](references/uda.md) |
| Wallet assets | Lukka-UDA | `get_address_assets` | [uda](references/uda.md) |
| Wallet transactions | Lukka-UDA | `get_address_transactions` | [uda](references/uda.md) |
| Transaction detail | Lukka-UDA | `Get_transaction_for_txhash` | [uda](references/uda.md) |
| Newest block | Lukka-UDA | `Get_newest_block_blockheight_v2` | [uda](references/uda.md) |
| Block detail | Lukka-UDA | `get_block` | [uda](references/uda.md) |
| Implied forward rates | Lukka-Analytics | `getImpliedRates` | [analytics](references/analytics.md) |
| OTC FX implied rates | Lukka-Analytics | `getImpliedRatesOtcFx` | [analytics](references/analytics.md) |
| Volatility surface | Lukka-Analytics | `getImpliedVolatilitySurface` | [analytics](references/analytics.md) |
| Digital asset news / articles | Lukka-News | `getNews` | [news](references/news.md) |
| News by sentiment | Lukka-News | `getNews` (with `sentiment` param) | [news](references/news.md) |
| Daily news summary / digest | Lukka-News | `getDailySummaries` | [news](references/news.md) |
| Regulatory / security news | Lukka-News | `getNews` (with `primaryCategory` filter) | [news](references/news.md) |
| Prediction markets | Lukka-Predmar | `GetPredictionMarkets` | [predmar](references/predmar.md) |
| Prediction market news | Lukka-Predmar | `GetMarketNews` (NOT Lukka-News `getNews`) | [predmar](references/predmar.md) |
| Market sentiment / overview | Lukka-Predmar | `GetMarketOverview` / `GetMarketRecap` | [predmar](references/predmar.md) |
| Similar markets | Lukka-Predmar | `GetSimilarMarkets` | [predmar](references/predmar.md) |
| Prediction classification | Lukka-Predmar | `GetPMCSstats` / `GetPredictionMarketsTaxonomy` | [predmar](references/predmar.md) |

## Decision Tree

```
Price / market data?       -> Lukka-Pricing
What is asset X?           -> Lukka-RefData (list_assets with assetCode filter)
Find assets by trait?      -> Lukka-RefData + client-side filter
AML / compliance?          -> Lukka-AML (report-only surface)
On-chain raw data?         -> Lukka-UDA
News / articles / headlines?-> Lukka-News
Prediction markets?        -> Lukka-Predmar
Derivatives match?         -> Lukka-Pricing (get_derivative_source_details)
Implied rates / curves?    -> Lukka-Analytics (getImpliedRates)
Vol surface?               -> Lukka-Analytics (getImpliedVolatilitySurface)
OTC FX rates?              -> Lukka-Analytics (getImpliedRatesOtcFx)
```

## Critical Patterns

**Pair format**: `BASE-COUNTER` - e.g. `XBT-USD`, `ETH-USD`, `SOLN-USD`. Not `BTC-USD`.

**Intervals in MILLISECONDS** (not seconds): `1000`, `5000`, `30000`, `60000`, `300000`, `1800000`, `3600000`, `86400000` (1s..1day, default `60000`). Hourly (`3600000`) works directly on `get_historical_prices` - no need for `get_candles`.

**Response modes** (pricing, refdata):
- Default = summary with stats
- `detailed: true` = paginated raw data
- `export: true` = CSV content

**Blockchain symbol case**:
- Lowercase for Lukka-UDA: `btc`, `eth`, `sol`, `trx`
- Uppercase for Lukka-AML report tools: `BTC`, `ETH`, `SOL`, `TRX`

**Source IDs** (Lukka-Pricing) - fixed high-level:
- `1000` = Lukka Prime EOD (also market cap)
- `2000` = Intraday, `2100` = Prime Global Intraday
- `3000` = Hourly
- `5001` / `5002` = Reference Rate / Median Reference Rate - available with Lukka Reference Rate access (request at https://lukka.tech/plugin); standard path: sourceId 1000 (EOD) / 2000 (Intraday)
- `5004` = Staking Rates, `5005` = MVWAP

Client-assigned (discover via `list_pricing_sources`, do NOT guess): exchanges `4001-4999`, index `6xxx`, custom `10xxx`.

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| "Invalid interval" | Seconds not ms | Use ms: `1000`, `5000`, `30000`, `60000`, `300000`, `1800000`, `3600000`, `86400000`. Hourly = `3600000` |
| Market cap wrong order | Default returns rank 1 first | Pass `order:"DESC"` explicitly for "top N" queries |
| `get_market_depth` 6.8 MB | Missing pair filters | `sourceIds` (comma-separated OK, e.g. `"4001,4002"` = one record per source) + `baseAssetCodes` + `counterAssetCodes` |
| `list_assets` 500 | Bare comma-separated `lukkaAssetIds`/`expandDetails` | Wrap value in square brackets: `[LA1,LA2]`, `[ALL]` |
| `get_latest_index_prices` 404 | Reference Rate access required | Available with Lukka Reference Rate access - request at https://lukka.tech/plugin. Standard path: `get_latest_prices({ sourceId: 2000 })` for realtime or `{ sourceId: 1000 }` for EOD |
| COINBASE not canonical | `get_entity_by_code("COINBASE")` does not resolve to the exchange | Use `CPRO` (Coinbase Pro), `CBSE`, `CPRM` (Coinbase Prime) |
| Wrong blockchain case | `eth` vs `ETH` | uda=lowercase, aml=UPPERCASE |
| Solana wrong code | Name lookups return an unrelated same-ticker ERC-20 token | Solana = `SOLN` (not SOL). Use known canonical codes |
| `generateAmlSimplifiedReport` wrong chain | Each tier supports specific chains | Use the tier matching your chain: SCORING (all chains), STANDARD/ENHANCED (30 chains), SIMPLIFIED (80 chains) - see [aml](references/aml.md) for full lists |
| `getNews` date filter rejected | Date not full ISO 8601 | Use full ISO 8601: `2026-04-01T00:00:00Z` |
| `generateAmlStandardReport` context overflow | 325 KB+ payload | Pipe through `large-data-processing/process_aml_report.py` |
| `get_address_transactions` context overflow | limit=1000 verbose | Preview at limit<=100, bulk via `process_transactions.py` |
| `get_core_asset_scores` empty | Missing fromDate+toDate | Pass both ISO 8601 parameters |

---

**Deep-dive references**: [pricing](references/pricing.md) - [refdata](references/refdata.md) - [analytics](references/analytics.md) - [aml](references/aml.md) - [uda](references/uda.md) - [news](references/news.md) - [predmar](references/predmar.md)
