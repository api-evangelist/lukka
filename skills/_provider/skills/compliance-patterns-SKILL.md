---
name: compliance-patterns
description: >-
  AML and VASP due diligence workflow patterns for digital asset compliance against
  the Lukka hosted MCP surface. Use when user asks about: AML risk, OFAC, VASP,
  due diligence, compliance review, address investigation, exchange compliance,
  or regulatory status.
argument-hint: "<address, exchange code, or compliance question>"
---

# Compliance Patterns

**STANDALONE**: This skill provides AML/VASP compliance workflow patterns, risk categories, and exchange code conventions useful for structuring compliance reviews. **SUPERCHARGED**: When Lukka-AML, Lukka-RefData, and Lukka-UDA MCP servers are connected, it orchestrates live AML scoring, counterparty graph analysis, and VASP due diligence.

Workflow patterns against the hosted Lukka MCP servers: **Lukka-AML**, **Lukka-RefData**, **Lukka-UDA**. All tool names use the production wire format `mcp__<Server>__<group>___<tool>`.

> The hosted surface provides report generation (scoring, standard, enhanced, simplified) plus UDA on-chain data via Lukka-UDA.

## Pattern 1: Address Investigation (default)

```
1. mcp__Lukka-UDA__UDA___get_address_balance({ blockchain: "eth", address })         -> current holdings
2. mcp__Lukka-UDA__UDA___get_address_assets({ blockchain: "eth", address })          -> token coverage
3. mcp__Lukka-UDA__UDA___get_address_transactions({ blockchain, address,
                                                    verbose:"1", limit:"100",
                                                    get_newest:"1" })                -> recent activity preview
4. mcp__Lukka-AML__AML___generateAmlScoringReport({ address, address_type: "ETH" })    -> light risk assessment
```

**Blockchain case**: Lukka-UDA uses lowercase (`eth`), Lukka-AML uses UPPERCASE (`ETH`).

## Pattern 2: Deep AML Review (on request)

Always pipe Standard / Enhanced reports through the `large-data-processing` skill's `process_aml_report.py` - raw responses are 300-500 KB+ and overflow context.

```
1. cp "${CLAUDE_PLUGIN_ROOT}/skills/large-data-processing/references/process_aml_report.py" /tmp/
2. mcp__Lukka-AML__AML___generateAmlStandardReport({ address, address_type: "ETH" })   -> 325 KB report
3. Save raw JSON to /tmp/aml_report.json
4. python3 /tmp/process_aml_report.py /tmp/aml_report.json                            -> 2-3 KB summary
5. (optional) mcp__Lukka-AML__AML___generateAmlEnhancedReport                         -> deepest detail (same pipe)
```

## Pattern 3: Counterparty Exposure

For counterparty exposure, use `generateAmlStandardReport` / `generateAmlEnhancedReport`
(pipe through `process_aml_report.py`) - their payloads carry counterparty reasons with USD
values. For raw address assets, use `mcp__Lukka-UDA__UDA___get_address_assets`.

## Pattern 4: VASP / Exchange Due Diligence

Use canonical Lukka exchange codes. **Never** `COINBASE` - it is not a canonical code and does not resolve to the exchange. Use `CPRO` (Coinbase Pro), `CBSE` (Coinbase parent), `CPRM` (Coinbase Prime), `KRAK` (Kraken), `BINA` (Binance), `GMNI` (Gemini), etc.

```
# Build exchange profiles from get_entity_by_code + list_vasps + get_asset_mappings.
1. mcp__Lukka-RefData__refdata___get_entity_by_code({ code: "CPRO" })                 -> basic entity info
2. mcp__Lukka-RefData__refdata___list_vasps({ mappingStatus: "Active" })              -> VASP data, licenses
   # filter client-side for lukkaEntityCode == CPRO (entityCode filter not reliably honored)
3. mcp__Lukka-RefData__refdata___get_asset_mappings({ entityCode: "CPRO",
                                                       mappingStatus: "Active" })    -> listed assets + supportedFunctionList
4. Per row: mcp__Lukka-RefData__refdata___get_asset_by_id({ assetLid: "{lukkaAssetId}" })
             -> inspect assetDetails.potentiallySuspicious / potentiallyMemeCoin
```

To find suspicious listings, iterate `get_asset_mappings` results and inspect `potentiallySuspicious` / `potentiallyMemeCoin` flags client-side via `get_asset_by_id`.

## AML Report Depths

| Tool | Payload | Context impact | When to use |
|------|---------|----------------|-------------|
| `generateAmlScoringReport` | small | safe | Default / light screening. Available on all supported chains |
| `generateAmlStandardReport` | 300-500 KB | MUST pipe | Investigation. Available on 30 chains (see below) |
| `generateAmlEnhancedReport` | 500 KB+ | MUST pipe | Deep dive / escalation. Available on 30 chains (see below) |
| `generateAmlSimplifiedReport` | small | safe | Light report. Available on 80 additional chains (see below) |

### Report tier availability by chain

Use SCORING for a light check on any chain. For a full or deep report use STANDARD/ENHANCED on the chains listed. SIMPLIFIED covers the additional chains listed below.

- **SCORING** - available on all supported chains (light default).
- **STANDARD / ENHANCED** - available on: `ADA, AVAX, BCH, BSV, BTC, CC, DASH, DOGE, DOT, DOT_AH, ETC, ETH, FIL, ICP, IOTA_MAINNET, IOTAEVM, LTC, MATIC, NEAR, NEO, RSK, SOL, TRX, VET, XDAI, XLM, XRP, XTZ, ZEC, ZKS`
- **SIMPLIFIED** - available on: `ALGO, APT, ARBITRUM, ATOM, AZERO, BASE, BNB, BSC, CANTO, CELO, CHZ, CKB, COTI, CRO, CRONOS, DCR, DGB, DOGECHAIN, ECS, EGLD, EOS, ETHW, EVMOS, FET, FLOW, FTM, FUSE, GLMR, GRAM, HASH, HBAR, HT, IMX, IOTA, IOTX, KAVA, KCS, KDA, KLAY, KSM, LINEA, LSK, LUNA, LUNC, MINA, MNT, MOVR, NEO3, ONE, ONT, OP, PALM, PLS, QTUM, REEF, RON, ROOT, RUNE, SDN, SEI, SMR, STX, SUI, TAO, THETA, TLOS, TON, VIC, WAN, WAVES, WAX, XDC, XEC, XEM, XNO, XVG, XYM, ZEN, ZIL, ZKEVM`

## AML surface scope

The hosted Lukka-AML surface provides **report generation** only: `generateAmlScoringReport` (light, any chain), `generateAmlStandardReport` / `generateAmlEnhancedReport` (30 chains, pipe through the processor), `generateAmlSimplifiedReport` (80 additional chains). Reports are stateless (no persistence between calls).

For needs outside report generation:
- Sanctions/OFAC screening: read the scoring report's sanctions flags, or use an external OFAC list.
- Bulk checks: call `generateAmlScoringReport` in parallel per address.
- Address assets / raw transactions: use the Lukka-UDA tools (`get_address_assets`, `get_address_transactions`).

## Key Rules

1. **Default AML depth is scoring**. Escalate to Standard / Enhanced only on explicit user request, and always with the processor pipe.
2. **Blockchain case**: Lukka-UDA=lowercase, Lukka-AML=UPPERCASE.
3. **Canonical exchange codes**: `CPRO`, `CBSE`, `CPRM`, `KRAK`, `BINA`, `GMNI`. Never `COINBASE`.
4. **Exchange profiles**: build from `get_entity_by_code` + `list_vasps` + `get_asset_mappings`.
5. **Address assets**: use `mcp__Lukka-UDA__UDA___get_address_assets`.
