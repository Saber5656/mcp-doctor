# ADR-004: Offline by default; `--online` opt-in registry checks; zero telemetry

- Status: accepted (2026-07-11, approved by owner)

## Context

Registry lookups (package existence, deprecation, typosquat similarity) add real
diagnostic value but transmit the user's server-package inventory to an external
service and make results network-dependent. A security/diagnostic tool must be
predictable about what leaves the machine.

## Decision

1. Default scan performs zero outbound network I/O.
2. `--online` enables exactly one endpoint class: `https://registry.npmjs.org/<name>`
   (abbreviated metadata). Only package names extracted from configs are sent.
3. No telemetry, no update checks, no crash reporting — ever, in any mode. Adding any
   new outbound endpoint requires a new ADR.
4. Note: under `--connect`, spawned servers may themselves use the network (e.g. `npx`
   downloading the package). That is the server's own documented behavior, disclosed in
   the consent preview, and distinct from mcp-medic's own traffic.

## Consequences

- Offline-default reduces detection power in the default mode (MM7xx rules inactive);
  acceptable — text output hints that `--online` exists when npx-based servers are seen.
- Privacy posture is documentable in one sentence and testable (E2E asserts no sockets
  opened in default mode).
