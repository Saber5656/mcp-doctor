# Research: Prior Art and Differentiation

Status: verified 2026-07-11 (npm registry, GitHub).
Purpose: establish what already exists, why mcp-medic is still worth building, and which
design choices differentiate it. Material here justified the rename decision recorded in
`docs/decisions/ADR-001-product-name.md`.

## Direct competitors

### 1. `mcp-doctor` (npm) — Crooj026/mcp-doctor

The name collision that forced our rename.

| Attribute | Value |
|---|---|
| npm | `mcp-doctor` 0.1.1 (published 2026-01, 2 releases) |
| GitHub | Crooj026/mcp-doctor — **1 star, 0 forks** (checked 2026-07-11) |
| Runtime | Node.js 18+ |
| Clients | Claude Desktop, Cursor, VS Code, Claude Code, Windsurf |
| Static checks | JSON syntax (trailing commas etc.), command existence/executability, missing/empty env vars, hardcoded-secret warnings |
| Dynamic checks | Yes — **runs every server by default** (handshake test, 10 s timeout), opt-out via `--skip-health` |
| Fix | No (planned) |
| CLI | `mcp-doctor [check] [--skip-health] [--file <path>]` |

Assessment: same one-line concept, minimal execution. No Codex/Zed/Gemini support, no
cross-client duplicate analysis, no machine-readable report, no exit-code contract, no
registry checks, no redaction guarantees, and a **run-configured-commands-by-default**
consent model that we consider a security anti-pattern for a diagnostic tool
(a tampered or malicious config entry would be executed by the very tool the user runs
to investigate it).

### 2. `mcp-checkup` (npm)

| Attribute | Value |
|---|---|
| npm | 0.1.9, last modified 2026-04 |
| Form | An **MCP server** (not a CLI) that analyzes your MCP setup |
| Focus | Token costs, bloated tool schemas, duplicate tools, optimization tips |

Assessment: adjacent niche (context-window optimization), not a configuration doctor.
Non-competing; possibly complementary. Its existence took the `mcp-checkup` name.

### 3. `@modelcontextprotocol/inspector`

Official interactive debugging UI (0.22.0). Targets **server developers** inspecting a
single server they are building (web UI, manual interaction). Not a batch, cross-client,
CI-friendly diagnostic. mcp-medic targets **users of many servers**; different job.

### 4. Client built-ins

- `claude mcp list` / `/mcp`: per-client, shows connection status but no static analysis, no security rules, no cross-client view.
- `codex mcp list`, `gemini mcp`: same limitation — each sees only its own silo.
- VS Code `chat.mcp.discovery`: imports other clients' servers rather than diagnosing them.

The cross-client, security-focused, machine-readable diagnostic slot is genuinely open.

## Naming ecosystem check (2026-07-11)

| npm name | Status |
|---|---|
| `mcp-doctor` | taken (competitor above) |
| `mcp-checkup` | taken (tool above) |
| **`mcp-medic`** | **available** → chosen (ADR-001) |
| `mcp-clinic`, `mcp-triage`, `mcp-doctor-cli`, `mcp-diag` | available (fallbacks) |

`doctor`-style CLIs are a well-understood mental model (`brew doctor`, `expo-doctor` 1.20,
`yeoman-doctor`, `@react-native-community/cli-doctor`), which validates the product shape:
zero-config binary that inspects an installation and prints actionable findings.

## Differentiation summary (drives v1 scope)

| Axis | Competitor state | mcp-medic v1 position |
|---|---|---|
| Client coverage | ≤5 clients, no Codex/Zed/Gemini | 8 clients + declarative custom adapter |
| Consent model | dynamic checks run by default | **static by default; `--connect` opt-in gate** (ADR-003) |
| Security rules | one "hardcoded secret" warning | rule catalog: secrets, file permissions, non-TLS URLs, unpinned `npx -y`, argv-visible secrets, shell-wrapper commands, `trust: true`, deprecated SSE, reserved names |
| Cross-client analysis | none | duplicate/shadowing/version-divergence rules over a normalized inventory |
| Output | human text only | text + stable versioned JSON schema + Markdown; documented exit codes for CI |
| Privacy | not stated | offline by default; `--online` opt-in registry checks; no telemetry (ADR-004) |
| Redaction | not stated | secrets never emitted on any output path; terminal-escape sanitization of untrusted config strings |
| Registry intelligence | none | opt-in: package existence, deprecation, typosquat similarity |

## Risks noted

1. Low barrier to entry — the competitor could catch up; our durable moats are the
   rule catalog depth, the adapter matrix, and the trustworthy security posture.
2. Client config formats churn (VS Code and Windsurf have both changed paths/branding
   within 12 months). Mitigation: research doc + adapter contract tests + per-adapter
   "format verified on" dates.
3. `npx mcp-medic` vs `npx mcp-doctor` discoverability: mitigated by keywords, README
   positioning, and the fact that the incumbent has ~zero adoption.
