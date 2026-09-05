# ADR-006: MIT license

- Status: accepted (2026-07-11, proposed as default, no owner objection)

## Context

The project is public OSS. Ecosystem norm: the MCP TypeScript SDK, most MCP servers,
and the adjacent tools are MIT. Apache-2.0 (patent grant) was considered.

## Decision

MIT. Maximal adoption/compatibility with the surrounding ecosystem; the project has no
patent-sensitive surface that would justify Apache-2.0's extra friction.

## Consequences

- `LICENSE` file (MIT, copyright line: `2026 mcp-medic contributors` — avoids
  publishing personal names) ships in the scaffolding issue; `package.json`
  `license: "MIT"`.
- Dependencies must remain MIT/ISC/BSD/Apache-2.0-compatible (checked in CI via
  license audit step if a copyleft dep ever appears — none planned).
