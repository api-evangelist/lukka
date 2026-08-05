---
name: digital-asset-market-knowledge
description: >-
  Domain knowledge for digital asset market data queries. Loaded as base context for
  all Lukka tools. Covers pair formats, source IDs, interval conventions,
  blockchain symbols, top asset codes, and response modes.
argument-hint: ""
---

# Digital Asset Market Knowledge

**STANDALONE**: This skill provides digital asset market conventions (pair formats, source IDs, interval units, blockchain symbols, asset codes) useful for any digital asset data work. **SUPERCHARGED**: When Lukka MCP servers are connected, these conventions prevent common API errors (wrong intervals, wrong pair codes, wrong blockchain case).

Base domain knowledge for working with Lukka's digital asset data infrastructure.

## Pair Format

Lukka uses canonical asset codes, NOT white paper codes:
- Bitcoin: `XBT` (not BTC) - LID `LA6EV2NKQ95`
- Ethereum: `ETH` is canonical (LID `LA8VRAGEMT1`, name "Ether") - `XET` is NOT the canonical code
- Solana: `SOLN` (not `SOL` - `SOL` is an unrelated ERC-20 token sharing the ticker)
- Pairs: `XBT-USD`, `ETH-USD`, `SOLN-USD`, `ADA-USD`

## Intervals (MILLISECONDS)

| Duration | Milliseconds | Common mistake |
|----------|-------------|----------------|
| 1 minute | 60000 | 60 (seconds) |
| 1 hour | 3600000 | 3600 (seconds) |
| 1 day | 86400000 | 86400 (seconds) |

**`get_historical_prices`**: 1min (60000), 1hr (3600000), 1day (86400000) all work.
**`get_candles`**: 1min, 1hr (3600000), 1day. Use candles when you need OHLCV instead of a single price.

## Source IDs

| ID | Name | Use case |
|----|------|----------|
| 1000 | Lukka Prime EOD | End-of-day prices, market cap |
| 2000 | Intraday | Real-time/intraday prices |
| 2100 | Lukka Prime Global Intraday | Intraday candles |
| 3000 | Hourly Prices | Hourly aggregated prices (NOT the reference rate) |
| 5001 | Lukka Reference Rate | Reference Rate (may return 404 - use the fallback below) |
| 5002 | Lukka Median Reference Rate | Median Reference Rate |
| 4001-4999 | Exchanges | Per-exchange data |

Discover exchange sourceIds: `list_pricing_sources({})`

## Blockchain Symbols

| Context | Format | Examples |
|---------|--------|----------|
| Lukka-UDA (on-chain) | lowercase | `btc`, `eth`, `sol`, `trx` |
| Lukka-AML (compliance) | UPPERCASE | `BTC`, `ETH`, `SOL`, `TRX` |

## Top Assets by Lukka Code

| Lukka Code | Asset | White Paper Code |
|------------|-------|------------------|
| XBT | Bitcoin | BTC |
| ETH | Ethereum | ETH |
| SOLN | Solana | SOL |
| ADA | Cardano | ADA |
| XRP | XRP | XRP |
| BNB | BNB | BNB |
| PDT | Polkadot | DOT |
| AVAX | Avalanche | AVAX |
| MATIC | Polygon | MATIC |
| LINK | Chainlink | LINK |
| UNISW | Uniswap | UNI |

Canonical codes: Ethereum = `ETH` (LID LA8VRAGEMT1), Polkadot = `PDT` (LA8FGEG06T7), Uniswap = `UNISW` (LA79U4XBU94). The entity/whitepaper ticker (DOT, UNI) resolves to these via `get_asset_mappings(entityAssetCode:...)`.

**TRAP: "SOL" in Lukka is NOT Solana** -- Lukka code for Solana is `SOLN`. Pair: `SOLN-USD`.
To resolve on the hosted surface, use `get_asset_mappings({ entityAssetCode: "SOL", mappingStatus: "Active" })` and collapse LID groups - the real Solana asset dominates by mapping count. Name lookups resolve through RefData mappings.

## Response Modes

| Mode | Parameter | Returns |
|------|-----------|---------|
| Summary (default) | none | Aggregated stats + samples |
| Detailed | `detailed: true` | Paginated raw data (100/page) |
| Export | `export: true` | CSV content |

## Pagination

- **Lukka-Pricing**: Cursor-based (`cursor` + `hasMore`) on some endpoints; `get_latest_market_caps` returns `{data:[...]}` with no cursor
- **Lukka-RefData**: Offset-based (`offset`/`limit`, default 100, max 1000)
- Name lookups resolve through RefData mappings (alias map -> get_asset_mappings -> get_asset_by_id)

## TOON Format

Many MCP servers return data in TOON format (Token-Optimized Object Notation):
- Header: `arrayName[count]{field1,field2,...}:`
- Rows: indented CSV lines
- 30-60% token savings vs JSON
