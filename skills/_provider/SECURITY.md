# Security

## Reporting Vulnerabilities

If you discover a security vulnerability in this plugin, report it
privately:

- **Email**: plugin-support@lukka.global
- **Subject line**: `[PLUGIN-SECURITY] Brief description`


Do NOT open public issues for security vulnerabilities.

## Scope

**Plugin (report with [PLUGIN-SECURITY]):**

- Plugin code: commands, skills, agents, hooks, processors
- `.mcp.json` configuration and OAuth flow
- Audit-logging hook and logged data

**Hosted MCP servers (report with [MCP-SECURITY]):**

- aml.mcp.lukka.tech, uda.mcp.lukka.tech, pricing.mcp.lukka.tech
- refdata.mcp.lukka.tech, analytics.mcp.lukka.tech, news.mcp.lukka.tech
- predmar.mcp.lukka.tech

**Claude Code / Anthropic platform:**

- https://www.anthropic.com/responsible-disclosure-policy

## Authentication

The seven Lukka OAuth servers authenticate via OAuth 2.0 with PKCE. No client ID
or secret is embedded in the plugin: each server entry in `.mcp.json` is only a
URL, and Claude Code discovers the authorization server automatically (RFC 9728
protected-resource metadata at `{server}/.well-known/oauth-protected-resource`).
The discovered identity provider issues a dynamic client via Client ID Metadata
Document (CIMD) - no static credential is stored anywhere in the plugin. Tokens
are cached locally in `~/.claude.json` by Claude Code. `Lukka-Predmar` is
OAuth-gated like the other six servers.

## Data Handling

The plugin keeps a local audit log of hosted MCP calls. It is local-only and
user-owned: the hook performs no network calls and never transmits data off the
machine. Records hold operational metrics only (size, timing, tool name) - never
payload content. Files live under the user's home directory and can be deleted at
any time.

The audit-logging hook (`hooks/usage_logger.py`) appends MCP tool calls to
`~/.claude/logs/lukka-usage-YYYY-MM.jsonl`.

**Logged fields:**

| Field | Description |
|-------|-------------|
| `ts` | ISO 8601 UTC timestamp |
| `tool` | MCP tool name (e.g. `mcp__Lukka-Pricing__pricing___get_latest_prices`) |
| `input_size` | Request payload size in bytes |
| `output_size` | Response payload size in bytes |
| `duration_ms` | Round-trip duration in milliseconds |
| `success` | Boolean success flag |
| `session` | Claude Code session identifier (opaque string, not user-identifying) |

**Not logged:** wallet addresses, query parameters, response content, user
emails, account IDs.

Logs rotate monthly by filename convention. Files are local to your machine.
You may delete them at any time.

**Opt-out:** Remove `hooks/hooks.json` from the plugin directory.

## Temporary File Processing

The `large-data-processing` skill writes intermediate files (AML reports,
transaction data) to `/tmp/` for processing. These files may contain
sensitive data (wallet addresses, risk scores, counterparty graphs). Files
are used ephemerally during command execution. On shared or multi-tenant
systems, ensure `/tmp/` has appropriate access controls.
