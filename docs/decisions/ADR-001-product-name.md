# ADR-001: Product name is `mcp-medic`

- Status: accepted (2026-07-11, approved by owner)
- Deciders: repository owner; researched by design agent

## Context

The repository was created as `mcp-doctor`. Research (2026-07-11, see
`docs/research/prior-art.md`) found the npm package name `mcp-doctor` is already
published (v0.1.1, Crooj026/mcp-doctor) with a near-identical concept, and
`mcp-checkup` is also taken. The incumbent has ~zero adoption (1 star, 0 forks), but
npm unscoped names are first-come. For an npx-first CLI, the npm name *is* the product
name; a scoped name (`@scope/mcp-doctor`) or a `-cli` suffix would hurt discoverability
or invite fork confusion.

Alternatives considered:
- A) `mcp-doctor-cli` (npm) + keep repo name — minimal change; search-results collision
  and "is this a fork?" ambiguity remain.
- B) **`mcp-medic` unified rename — chosen.**
- C) `@saber5656/mcp-doctor` — longest npx invocation, weakest discoverability.

## Decision

The product, npm package, binary, and repository are all named **`mcp-medic`**
(npm availability verified 2026-07-11). All documentation and issues use `mcp-medic`.

## Consequences

- GitHub repository `Saber5656/mcp-doctor` must be renamed to `mcp-medic` and the
  local directory renamed accordingly — **manual owner action** (tracked as a handoff
  item in `docs/ISSUE_PLAN.md`).
- The npm name should be reserved early (placeholder 0.0.1 publish or trusted-publisher
  setup) — manual owner action, since credentials are owner-held.
- Medical metaphor (`medic`) keeps the `brew doctor`-style mental model without the
  collision. Fallback names if `mcp-medic` is lost before reservation: `mcp-clinic`,
  `mcp-triage`, `mcp-diag`.
