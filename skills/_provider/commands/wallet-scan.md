---
description: Wallet analysis with AML risk, balance, assets, and transaction preview
argument-hint: "<address> <blockchain, e.g. eth>"
allowed-tools:
  - mcp__Lukka-AML__AML___generateAmlEnhancedReport
  - mcp__Lukka-AML__AML___generateAmlScoringReport
  - mcp__Lukka-AML__AML___generateAmlSimplifiedReport
  - mcp__Lukka-AML__AML___generateAmlStandardReport
  - mcp__Lukka-UDA__UDA___get_address_assets
  - mcp__Lukka-UDA__UDA___get_address_balance
  - mcp__Lukka-UDA__UDA___get_address_transactions
---

# /wallet-scan

> See [CONNECTORS.md](../CONNECTORS.md) for tool details.

## Usage

/wallet-scan $ARGUMENTS

Parse $ARGUMENTS for address and blockchain symbol. Remember: Lukka-UDA uses lowercase (`eth`, `btc`), Lukka-AML uses uppercase (`ETH`, `BTC`).

> **UDA vs AML chain coverage differ.** UDA address/tx endpoints (steps 1-3) serve: btc, bch, ltc, doge, dash, eth, avax, xrp, xlm, near, dot (+ vet balance-only). `sol`, `trx`, `matic`, `ada` return "Unsupported crypto symbol" on UDA even though AML (step 4) DOES score them. On a chain where AML scores but UDA address/tx endpoints do not serve (e.g. sol, trx), skip steps 1-3, note "on-chain data for {chain} is provided via the AML report", and run the AML scoring/standard/enhanced report only. See `skills/lukka-data-routing/references/uda.md` for the full per-endpoint matrix.

## Flow (context-safe sequence)

1. **Balance**: `mcp__Lukka-UDA__UDA___get_address_balance({ blockchain: "{lc}", address: "{address}" })` [~~onchain]
2. **Assets**: `mcp__Lukka-UDA__UDA___get_address_assets({ blockchain: "{lc}", address: "{address}" })` [~~onchain]
3. **Transaction preview** (always cap at 100 to stay within context):
   `mcp__Lukka-UDA__UDA___get_address_transactions({ blockchain: "{lc}", address: "{address}", verbose: "1", limit: "100", get_newest: "1" })` [~~onchain]
4. **Light AML (default)**: `mcp__Lukka-AML__AML___generateAmlScoringReport({ address: "{address}", address_type: "{UC}" })` [~~aml]
   - **Parameter is `address_type`** (e.g. `"ETH"`, `"BTC"`), NOT `blockchain`. Same for `generateAmlStandardReport` / `Enhanced`.
   - Returns C-Score (0-100) + risk_level (LOW/MEDIUM/HIGH) + top factors. Fits in context.
   - To reuse an existing report: `--reuse-report {report_id}` skips re-scoring

## On request: deeper analysis

### Full transaction scan (>100 txs)

Before calling with large limit, deploy the processor so the raw response never enters context. See `large-data-processing` skill.

```
cp "${CLAUDE_PLUGIN_ROOT}/skills/large-data-processing/references/process_transactions.py" /tmp/
# Call MCP with limit:"1000", save raw JSON to /tmp/txs.json
python3 /tmp/process_transactions.py /tmp/txs.json --stats --top-senders 10 --dust-threshold 0.0001
# Then selectively decode flagged hashes via Get_transaction_for_txhash
```

### Full AML report (325 KB+ - mandatory processor pipe)

```
cp "${CLAUDE_PLUGIN_ROOT}/skills/large-data-processing/references/process_aml_report.py" /tmp/
# Call mcp__Lukka-AML__AML___generateAmlStandardReport, save raw to /tmp/aml_report.json
python3 /tmp/process_aml_report.py /tmp/aml_report.json
```

For maximum depth: `mcp__Lukka-AML__AML___generateAmlEnhancedReport` (500 KB+) - same processor workflow.

### Counterparty exposure (on demand)

> **Note**: for counterparty exposure data, use `generateAmlStandardReport` (with processor pipe).

> **Report tier availability**: Use SCORING for a light check on any chain. For a full or deep report use STANDARD/ENHANCED on these 30 chains: `ADA, AVAX, BCH, BSV, BTC, CC, DASH, DOGE, DOT, DOT_AH, ETC, ETH, FIL, ICP, IOTA_MAINNET, IOTAEVM, LTC, MATIC, NEAR, NEO, RSK, SOL, TRX, VET, XDAI, XLM, XRP, XTZ, ZEC, ZKS`. SIMPLIFIED covers the additional 80 chains: `ALGO, APT, ARBITRUM, ATOM, AZERO, BASE, BNB, BSC, CANTO, CELO, CHZ, CKB, COTI, CRO, CRONOS, DCR, DGB, DOGECHAIN, ECS, EGLD, EOS, ETHW, EVMOS, FET, FLOW, FTM, FUSE, GLMR, GRAM, HASH, HBAR, HT, IMX, IOTA, IOTX, KAVA, KCS, KDA, KLAY, KSM, LINEA, LSK, LUNA, LUNC, MINA, MNT, MOVR, NEO3, ONE, ONT, OP, PALM, PLS, QTUM, REEF, RON, ROOT, RUNE, SDN, SEI, SMR, STX, SUI, TAO, THETA, TLOS, TON, VIC, WAN, WAVES, WAX, XDC, XEC, XEM, XNO, XVG, XYM, ZEN, ZIL, ZKEVM`.
> **Do NOT call `generateAmlStandardReport` or `generateAmlEnhancedReport` without the processor** - responses exceed context window.

### Smart contract interaction detection

When UDA `get_address_transactions` returns `value=0` for a transaction with `input != '0x'`, this indicates a smart contract interaction (token transfer/swap). For decoded token amounts, follow up with `Get_transaction_for_txhash` using the tx_hash.

## Output

### Wallet Scan: {address}

**Compliance (AML Scoring)**
- Risk Level: {risk_level} (LOW/MEDIUM/HIGH from `cscore_section.risk_level`)
- C-Score: {cscore}/100 (from `cscore_section.cscore`)
- Top Risk Factors: {list from `factors`}

**Holdings**
- Native Balance: {amount} {symbol}
- Tokens: {count} contracts (by order of last interaction)

| # | Token Contract | Last Seen |
|---|---------------|-----------|

> USD values are not resolved by default (requires N*3 calls per token). Use `--full-enrichment` to resolve balances and prices for each token.

**Activity Preview** (last 100 txs)
- Total shown: {count}
- Date range: {first} to {last}
- Peak day: {date} ({count} txs)

**Heuristic Flags** (Claude analysis of AML factors + tx preview - not authoritative)
- {pattern description, e.g. "5 dust transactions from addresses matching mixer pattern"}
- Based on: AML risk factors + transaction preview pattern matching
- Disclaimer: heuristic only - for authoritative flagging use Standard/Enhanced AML report

**Next steps** (offer to user):
- `--full-enrichment`: resolve USD values per token (high call volume)
- Full transaction scan (deploy processor)
- Standard or Enhanced AML report (deploy processor)
- Tx flow graph
