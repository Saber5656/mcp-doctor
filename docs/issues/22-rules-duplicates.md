# Title

Duplicate & consistency rules MM301–MM304

# Summary

Implement the four cross-inventory rules that detect scope shadowing inside one
client, the same server configured in multiple clients, divergent duplicate
definitions, and version-pin divergence — over a normalized server-identity function.

# Context

DESIGN.md §10 (Duplicates table). This is a headline differentiator (prior-art doc):
no existing tool correlates MCP config across clients. Requires a precise identity
definition to avoid false "duplicates".

# Scope

- `src/rules/duplicates/identity.ts` (+ mm301–mm304 modules + barrel)
- `tests/unit/rules/duplicates/*.test.ts`

# Detailed Requirements

1. `identity.ts` — `serverIdentity(e: ServerEntry): string | null`:
   - stdio with pkgspec: `npm:<name>` / `pypi:<name>`. Package identity excludes
     version **and args**: MM304 compares versions inside an identity group, and
     differing args surface via MM303.
   - stdio without pkgspec: `cmd:` + normalized `command` (absolute paths via
     `path.normalize`; bare names as-is; no basename folding, which would be too
     loose) + ` ` + args joined with ` ` -- raw-command identity **includes** args,
     order preserved.
   - remote: `url:` + URL normalized (lowercase host, default ports stripped, no
     trailing slash, query preserved, **credentials stripped**).
   - Entries with `unresolvedVars` → null (identity unknowable; excluded from 3xx).
2. **MM301** (inventory, warn): same `client` + same `name` in >1 scope. Detail
   names both files. Precedence phrasing: **claude-code only** has documented
   precedence (local > project > user; collisions involving `managed` scope state
   "managed policy applies; precedence undocumented"); every other client states
   "same-name precedence is undocumented for this client — remove one definition to
   avoid ambiguity" (the research doc records no precedence for them; do not
   invent). One finding per (client, name) group; for claude-code, anchored at the
   losing definition's location, otherwise at the second occurrence.
3. **MM302** (inventory, info): same identity in ≥2 different clients — a compact
   inventory aid listing client:scope pairs. One finding per identity group.
4. **MM303** (inventory, warn): same identity, any of: differing `env` **key sets**
   (values never compared — they are secrets), differing args vectors, differing
   headers key sets. Detail is a keyset diff (e.g. `cursor defines API_BASE; codex does not`).
5. **MM304** (inventory, warn): same pkgspec `(ecosystem, name)` group, ≥2 distinct
   exact `version` pins (null/unpinned entries excluded — MM404's turf; npm and
   pypi packages with the same bare name never compare). Detail lists
   version-per-location; remediation: align to one pin.
6. All four rules must be O(n log n): one inventory pass fills three maps —
   `Map<identity, entries>` (MM302/MM303), `Map<client:name, entries>` (MM301),
   `Map<ecosystem:pkgname, entries>` (MM304).

# Acceptance Criteria

- [ ] Identity: `npx -y pkg@1.0.0` ≡ `npx pkg@2.0.0` (same identity → MM304 case);
      `https://h/mcp` ≡ `https://h:443/mcp/`; `https://u:p@h/mcp` ≡ `https://h/mcp`;
      pkgspec identity ignores args (`npx pkg --port 1` ≡ `npx pkg --port 2`) while
      raw-command identity includes them (`/srv/x --a` ≢ `/srv/x --b`) — all pinned
      by tests.
- [ ] MM301 fixtures: claude-code user+local same name → one warn naming
      winner=local; cursor user+project same name → one warn with the
      "precedence undocumented" phrasing.
- [ ] MM303 fires on env-keyset diff, silent on value-only diff (explicit test with
      different secret values, same keys).
- [ ] No 3xx findings for entries with unresolved vars.
- [ ] Group findings count: 3 clients sharing one identity → exactly one MM302.

# Validation

`npm test -- --run tests/unit/rules/duplicates`; reviewer verifies the claude-code
precedence phrase against research doc §2 and confirms no other client's finding
claims a documented precedence.

# Dependencies

19, 21 (pkgspec).

# Non-goals

- No cross-client sync/migration suggestions (v2), no probing, no value comparison of
  secrets ever.

# Design References

- DESIGN.md §10 Duplicates (MM3xx); research doc cross-client observation #9
