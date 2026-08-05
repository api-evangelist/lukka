---
name: large-data-processing
description: >-
  Process large MCP responses (>100KB) locally instead of loading into context.
  Covers market depth (6.8MB+), bulk address transactions (150KB+), block
  batch analysis (675KB+), and AML standard/enhanced reports (300-500KB+).
  Deploy Python processors to /tmp, pipe raw JSON through them, read concise
  output (~2-3KB) back into context.
argument-hint: "<path to JSON file, e.g. /tmp/depth.json>"
---

# Large Data Processing

**STANDALONE**: This skill provides Python processor scripts (stdlib only) that summarize large JSON files locally -- market depth, transactions, blocks, AML reports. Works on any JSON data matching the expected schemas. **SUPERCHARGED**: When Lukka MCP servers are connected, these processors prevent context overflow from oversized API responses (6.8MB market depth, 500KB AML reports).

**Problem**: Some MCP responses are too large for context (6.8MB market depth, 1000 decoded txs).
**Solution**: Save raw JSON to /tmp, process locally with Python scripts, read summary.

## When to Use

| Endpoint | Trigger | Size |
|---|---|---|
| `get_market_depth` without pair filter | Full exchange dump | 6.8MB, 1500+ pairs |
| `get_order_book_snapshot` (single pair) | L2 book snapshot | ~3MB EVEN WITH pair filter - always pipe |
| `get_derivative_source_details` | Derivative pair catalog | ~1.5MB (3800+ pairs) - grep for the target code, never load raw |
| `get_historical_derivative_trades` | Derivative trade prints | 300KB+ per 1h; `limit` is per-time-page NOT per-record, so it does NOT cap size. Use smallest window + pipe |
| `getImpliedRatesOtcFx` | OTC FX term structure | 400KB+; `limit` caps time-pages not rows (all tenors x all hours return). Narrow window + pipe |
| `get_asset_by_id` (hosted) | Full asset detail | ~54KB raw (hosted has no summary mode) - fits context but trim if batching |
| `get_address_transactions` verbose, limit>100 | Bulk wallet analysis | 150KB+ |
| `get_block` + batch decode | Block-level forensics | 675KB+ (135 txs) |
| `generateAmlStandardReport` | Full AML report | 300-500KB (ALWAYS pipe) |
| `generateAmlEnhancedReport` | Deepest AML report | 500KB+ (ALWAYS pipe) |
| Any MCP response | Response feels truncated or huge | >50KB |

## Workflow

### Step 1: Detect large response risk

Before calling, check if the query will return bulk data:
- `get_market_depth` without `baseAssetCodes`/`counterAssetCodes` = 6.8MB
- `get_order_book_snapshot` = ~3MB even with a single pair filter
- `get_derivative_source_details` = ~1.5MB (grep for the pair, never load raw)
- `get_historical_derivative_trades` / `getImpliedRatesOtcFx` = 300-400KB+; `limit` does NOT cap total size (it pages by time), so shrink the from/to window
- `get_address_transactions` with `limit: 1000` + `verbose: "1"` = 150KB+
- Batch decoding >50 transactions = 250KB+

### Step 2: Deploy processor script

Copy the appropriate script from this skill's references to /tmp. When invoked as a plugin skill, use `${CLAUDE_PLUGIN_ROOT}` to resolve the plugin install path:

```bash
# Market depth
cp "${CLAUDE_PLUGIN_ROOT}/skills/large-data-processing/references/process_market_depth.py" /tmp/

# Transactions
cp "${CLAUDE_PLUGIN_ROOT}/skills/large-data-processing/references/process_transactions.py" /tmp/

# Block data
cp "${CLAUDE_PLUGIN_ROOT}/skills/large-data-processing/references/process_block.py" /tmp/

# AML standard/enhanced report
cp "${CLAUDE_PLUGIN_ROOT}/skills/large-data-processing/references/process_aml_report.py" /tmp/
```

### Step 3: Call MCP and save raw output

Call the MCP tool with `format_response: false` or `detailed: true` where available.
Save the raw JSON response to /tmp:

```bash
# Example: save market depth response
echo '<raw_json>' > /tmp/depth.json
```

### Step 4: Process and read summary

```bash
# Market depth - top 20 most liquid pairs
python3 /tmp/process_market_depth.py /tmp/depth.json --top 20

# Transactions - full statistical summary
python3 /tmp/process_transactions.py /tmp/txs.json --stats --top-senders 10 --dust-threshold 0.0001

# Block analysis
python3 /tmp/process_block.py /tmp/block.json --stats --sample 5

# Multi-block batch (JSONL)
python3 /tmp/process_block.py /tmp/blocks.jsonl --range-stats

# AML report (Standard/Enhanced) - MANDATORY when calling those two tools
python3 /tmp/process_aml_report.py /tmp/aml_report.json
```

### Step 5: Follow up via MCP

Use the summary to identify specific items for deeper MCP queries:
- Decode flagged transactions: `mcp__Lukka-UDA__UDA___Get_transaction_for_txhash({ blockchain, transaction_hash })`
- Get specific pair depth: `mcp__Lukka-Pricing__pricing___get_market_depth({ sourceIds, baseAssetCodes, counterAssetCodes })`
- AML flagged counterparties: run `mcp__Lukka-AML__AML___generateAmlScoringReport` on each

## Script Reference

### process_market_depth.py

```
python3 process_market_depth.py <file.json> [options]
  --pair SOLN-USD          Filter specific pair
  --top N                  Top N pairs by depth
  --min-depth 50000        Min total depth (USD)
  --sort depth|spread|volume   Sort order (default: depth)
  --format table|csv|json  Output format (default: table)
```

Output: Exchange, Pair, Bid 1%, Ask 1%, Total Depth, Spread bps, Mid Price

### process_transactions.py

```
python3 process_transactions.py <file.json> [options]
  --stats                  Summary: count, volume, date range, errors
  --address 0x...          Target address (auto-detected if omitted)
  --top-senders N          Top N inbound addresses
  --top-recipients N       Top N outbound addresses
  --min-value 0.1          Min value filter
  --from-date 2026-01-01   Date filter (from)
  --to-date 2026-03-12     Date filter (to)
  --large-txs N            N largest txs (for MCP decode)
  --dust-threshold 0.0001  Dust detection threshold
  --format table|csv|json  Output format
```

Output: Total txs, date range, inflow/outflow, error rate, dust count, top senders/recipients, largest txs.

### process_block.py

```
python3 process_block.py <file.json> [options]
  --stats                  Block summary: tx count, gas, time
  --sample N               Random N tx hashes for decode
  --range-stats            Multi-block stats (JSONL input)
  --format table|csv|json  Output format
```

Output: Block number, timestamp, tx count, size, gas usage, base fee, sample hashes.

### process_aml_report.py

```
python3 process_aml_report.py <file.json> [options]
  --full-risks            Include all risk factors (not just top 10)
  --format table|json     Output format (default: table)
```

Output (~2-3 KB): report info (address, score, grade), top risk factors, flagged counterparties, exposure summary, monitoring flags (sanctions/darknet/ransomware/mixer counts), tx stats, recommendations.

**Always use this when calling** `mcp__Lukka-AML__AML___generateAmlStandardReport` or `mcp__Lukka-AML__AML___generateAmlEnhancedReport` - raw responses exceed context.

## Anti-patterns

- Do NOT load 6.8MB raw JSON into context - use this skill
- Do NOT decode all 135 block transactions one by one - sample first
- Do NOT call `get_market_depth` without pair filters unless you WANT full exchange data
- Do NOT call `generateAmlStandardReport` or `generateAmlEnhancedReport` without the processor - payload exceeds context
- Do NOT skip Step 2 (deploy) - scripts must exist in /tmp before use
