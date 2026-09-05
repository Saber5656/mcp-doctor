# ADR-002: TypeScript on Node.js, distributed via npm

- Status: accepted (2026-07-11, approved by owner)

## Context

The tool must parse many JSON/JSONC/TOML configs, speak MCP for dynamic probes, and be
trivially runnable by its target audience. Candidates: TypeScript/Node (npm/npx), Go
(single binary), Rust (single binary).

## Decision

TypeScript, ESM, Node.js `>=20`, published to npm as `mcp-medic`, executed primarily
via `npx mcp-medic`.

Rationale:
1. The official `@modelcontextprotocol/sdk` (1.29.x, TypeScript) provides the client
   and all transports — the probe engine needs no protocol reimplementation.
2. The target audience already runs Node: the dominant MCP server distribution
   mechanism is `npx`. A user with zero Node cannot have npx-based servers to diagnose.
3. Fastest path to the adapter/rule breadth that is the actual moat.

Trade-offs accepted: Node startup latency (~100–300 ms) is irrelevant for a diagnostic
tool; no single-binary distribution in v1 (Homebrew tap / SEA packaging can come later
without changing the architecture).

## Consequences

- Engines floor `node >= 20` (SDK requires ≥18; 20 gives stable `fetch`/AbortSignal).
  CI tests 20/22/24.
- Dependency policy is strict (see DESIGN §3.2) because npx cold-installs the tree.
- Version pins recorded 2026-07-11: sdk ^1.29, commander ^15, zod ^4.4, smol-toml ^1.7,
  jsonc-parser ^3.3, picocolors ^1.1, tsup ^8.5, vitest ^4.1.
