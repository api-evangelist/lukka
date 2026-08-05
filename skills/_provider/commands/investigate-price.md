---
description: Deep price investigation at specific timestamp with exchange breakdown
argument-hint: "<pair> [timestamp, e.g. SOLN-USD 2026-03-11T20:00:00Z]"
allowed-tools:
  - mcp__Lukka-UDA__UDA___get_block
  - mcp__Lukka-Pricing__pricing___get_candles
  - mcp__Lukka-Pricing__pricing___get_historical_prices
  - mcp__Lukka-Pricing__pricing___get_latest_prices
  - mcp__Lukka-Pricing__pricing___get_market_depth
  - mcp__Lukka-Pricing__pricing___get_market_variances
  - mcp__Lukka-Pricing__pricing___get_order_book_snapshot
  - mcp__Lukka-Pricing__pricing___list_pricing_sources
---

# /investigate-price

> See [CONNECTORS.md](../CONNECTORS.md) for tool details.

## Usage

/investigate-price $ARGUMENTS

Parse $ARGUMENTS for pair code and optional timestamp (default: latest).

## Flow

1. **Discover sources**: `mcp__Lukka-Pricing__pricing___list_pricing_sources({})` [~~pricing]
2. **Price at time**: `mcp__Lukka-Pricing__pricing___get_latest_prices({ sourceId: 1000, pairCodes: "{pair}", asOf: "{timestamp}" })` or `mcp__Lukka-Pricing__pricing___get_historical_prices({ pairCode: "{pair}", from, to, interval: 60000 })` [~~pricing] (sourceId 1000 = Lukka Prime EOD)
   - Note: price returned is the latest available *on or before* `asOf`. For exact minute-precision use `get_historical_prices({ interval: 60000 })`.
3. **Exchange variance**: `mcp__Lukka-Pricing__pricing___get_market_variances({ pairCode: "{pair}", from: "{ts-1h}", to: "{ts}" })` [~~pricing]
   - Provides per-exchange deviation from composite
4. **Candles per exchange**: For top exchanges, `mcp__Lukka-Pricing__pricing___get_candles({ pairCode: "{pair}", sourceId: {id}, from: "{ts-2h}", to: "{ts}", interval: 3600000 })` [~~pricing]
   - Provides per-exchange volume
   - **Merge step 3 + step 4** on `pairCode + sourceId`: variance fills Deviation column, candle fills Volume column
5. **Order book snapshot**: `mcp__Lukka-Pricing__pricing___get_order_book_snapshot({ sourceIds: "{single_sourceId}", pairCodes: "{pair}", asOf: "{timestamp}" })` [~~pricing]
   - Top exchange sourceId (e.g. 4002 Coinbase Pro or 4005 Binance), one at a time
6. **Market depth per top exchange**: `mcp__Lukka-Pricing__pricing___get_market_depth({ sourceIds: "{single_sourceId}", baseAssetCodes: "{base}", counterAssetCodes: "USD" })` [~~pricing]
   - **CRITICAL**: one sourceId per call + base/counter filters (avoids 6.8 MB full exchange dump)

## Output

### Price Investigation: {pair} at {timestamp}

**Lukka Prime Price**: ${price}

**Exchange Contributions**
| Exchange | Price | Share % | Volume | Deviation |
|----------|-------|---------|--------|-----------|

- Share % = `volume_exchange / total_volume_across_exchanges * 100` (computed from candles volume)

**Hourly Candles (leading 2h)**
| Time | Exchange | Open | High | Low | Close | Volume |
|------|----------|------|------|-----|-------|--------|

**Order Book Snapshot**
- Best Bid: ${bid} ({bid_size})
- Best Ask: ${ask} ({ask_size})
- Spread: ${spread} ({spread_bps} bps)

**Analysis**: {Describe deviation pattern, liquidity concentration, any outliers across exchanges}

> **Block-level analysis**: if the user wants on-chain context around the timestamp, use `mcp__Lukka-UDA__UDA___get_block` and the `large-data-processing` skill's `process_block.py`.
