# ADR-005: v1 is strictly read-only (no `fix` mode)

- Status: accepted (2026-07-11, approved by owner)

## Context

A `fix` command (remove duplicates, repair syntax, delete dead servers) is attractive
but writes to other applications' config files — the highest-risk operation the tool
could perform (backup strategy, atomic writes across 8 formats, client-running-while-
edited races, partial-write corruption). The owner chose to ship diagnosis first.

## Decision

1. v1 never opens any client config file for writing. The only file mcp-medic ever
   writes is the report given via `--output`.
2. Every finding must instead carry a concrete `remediation` string — where possible a
   copy-pasteable command (e.g. `claude mcp remove foo`) or an exact edit instruction
   with file path and JSON pointer.
3. `fix` is deferred to v2 and requires its own design + ADR (backup/undo model,
   per-client write strategies, dry-run).

## Consequences

- Trust story is simple and auditable ("this tool cannot break your setup").
- Remediation quality becomes a first-class acceptance criterion of every rule issue.
- Some user demand for auto-fix goes unmet in v1; README states the roadmap.
