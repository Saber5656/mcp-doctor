# Title

Core domain model and types (`src/core/model.ts`)

# Summary

Implement the shared type system — `ServerEntry`, `ConfigFileInfo`, `Inventory`,
`Finding`, `ProbeResult`, `Report`, enums — plus the `key` generation and transport
normalization helpers, exactly as specified in DESIGN.md §4 and §7.3.

# Context

Every other module imports these types; getting them right first prevents churn.
DESIGN.md §4 contains the normative TypeScript definitions; this issue turns them into
code with runtime helpers and tests.

# Scope

- `src/core/model.ts` (types + pure helpers)
- `tests/unit/core/model.test.ts`

# Detailed Requirements

1. Copy the interfaces from DESIGN.md §4 verbatim (names, fields, optionality).
   Deviations require editing DESIGN.md first — do not silently drift.
2. Implement `makeServerKey(client, scope, name, taken: Set<string>): string`:
   `${client}:${scope}:${name}`, appending `#2`, `#3`… on collision; mutates/records
   into `taken`.
3. Implement `normalizeTransport(raw: {type?: string; hasCommand: boolean; urlKey?: 'url'|'serverUrl'|'httpUrl'; client: ClientId}): Transport`
   implementing **every row** of the DESIGN.md §7.3 table — this helper is the single
   owner of transport normalization; adapters call it and only record bookkeeping
   flags (`clientSpecific.declaredType`, `transportAssumed`), never reimplement rows:
   - explicit `type` values: `stdio`/`http`/`sse`/`ws`; `streamable-http` → `http`;
     unrecognized declared type → `unknown` (adapter records `declaredType`; MM104
     keys off that flag)
   - gemini: `httpUrl` → `http`, `url` → `sse`
   - codex: `url` → `http`
   - cursor, windsurf: url-ish key present + no type → `http` (documented client
     behavior; caller records `transportAssumed: 'http'`)
   - url-ish present + no type + all other clients → `unknown`
   - no url-ish key + `hasCommand` → `stdio`
   The helper returns `Transport` only; *why* a value is `unknown` is tracked by the
   adapters' `clientSpecific` flags, not by this function.
4. Implement `severityRank(s: Severity): number` (error=3, warn=2, info=1) used by
   sorting and `--fail-on`.
5. All helpers pure; no I/O; no imports beyond types.
6. JSDoc on every exported symbol (issue 34 later re-exports them from
   `src/index.ts` as the library API; this issue does **not** touch `src/index.ts`).

# Acceptance Criteria

- [ ] Types compile under the strict tsconfig from issue 01 (including
      `exactOptionalPropertyTypes`).
- [ ] `makeServerKey` produces `a:user:x`, `a:user:x#2`, `a:user:x#3` for triples.
- [ ] `normalizeTransport` unit tests cover every row of the §7.3 table including the
      cursor/windsurf assumed-http row and the unrecognized-declared-type row
      (≥ 12 cases).
- [ ] `severityRank` unit-tested (error>warn>info ordering, exact values 3/2/1).
- [ ] No runtime dependency imports in `model.ts` (test asserts the built module's
      import list, or an ESLint `no-restricted-imports` override scoped to the file).
- [ ] Every exported symbol has JSDoc (reviewer checklist item; spot-checked).

# Validation

`npm run typecheck && npm test -- --run tests/unit/core/model.test.ts` green; reviewer
diffs interfaces against DESIGN.md §4 field-by-field.

# Dependencies

01.

# Non-goals

- No zod schemas (report schema is issue 31), no parsing, no adapter logic, no
  `src/index.ts` changes (issue 34 owns the public re-exports).

# Design References

- DESIGN.md §4 (data model), §7.3 (transport normalization)
