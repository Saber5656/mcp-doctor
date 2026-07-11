# Title

Adapter: Claude Desktop

# Summary

Implement `src/discovery/adapters/claude-desktop.ts`: discover and parse
`claude_desktop_config.json` into normalized `ServerEntry` values, with the visibility
note about UI-managed connectors/extensions.

# Context

Facts are normative in `docs/research/client-config-formats.md` §1. Claude Desktop is
the simplest adapter (single file, stdio-only, `mcpServers` key) and a good template
for the others.

# Scope

- `src/discovery/adapters/claude-desktop.ts` (+ registration in `adapters/index.ts`)
- Fixtures `tests/fixtures/homes/claude-desktop/{missing-file,valid-empty,valid-full,malformed,secrets-planted,jsonc-only,wrong-typed-fields}` + `expected.json` each
- `tests/unit/discovery/claude-desktop.test.ts` invoking the contract harness

# Detailed Requirements

1. `candidateFiles(ctx)` (scope `user`, format `json`):
   - darwin: `<appData()>/Claude/claude_desktop_config.json`
   - win32: `<appData()>/Claude/claude_desktop_config.json` (APPDATA-based)
   - linux: `<xdgConfig()>/Claude/claude_desktop_config.json` (best-effort; comment
     that Linux is not officially supported by Anthropic)
2. `parseFile` (records facts and `structuralIssues` — never `Finding` objects,
   per DESIGN §7.1):
   - Root must be an object; missing `mcpServers` key ⇒ zero entries (healthy —
     observed in the wild; see research doc "local evidence").
   - `mcpServers` not an object ⇒ `structuralIssues` entry
     (`kind: 'servers-key-not-object'`, pointer `/mcpServers`).
   - Each entry: `command` (string), `args` (string[], default `[]`),
     `env` (Record<string,string>, default `{}`). Wrong-typed fields follow fixed
     salvage rules: entry is emitted iff `command` is a string; non-array `args` →
     `[]`; non-object `env` (or non-string values within) → offending parts dropped
     to `{}`/omitted; every dropped/wrong-typed field appends a `structuralIssues`
     entry (`kind: 'field-wrong-type'`, pointer to the exact field). Entries without
     a string `command` are skipped with a `structuralIssues` entry
     (`kind: 'entry-not-usable'`).
   - `transport`: via `normalizeTransport` in all cases. Documented Claude Desktop
     entries have no url-ish/`type` keys and normalize to `stdio`; if `url`/`type`
     appear anyway, record `clientSpecific.declaredType` / pass `urlKey` through and
     accept the helper's outcome (`unknown` for this client) — do not invent client
     behavior.
   - `enabled`: always `true` (no disable mechanism in this file).
   - `source.pointer`: `/mcpServers/<name>` with RFC 6901 escaping of `<name>`
     (`~`→`~0`, then `/`→`~1`); line/col via `locate()` from issue 06.
3. `notes(ctx)`: exactly one note — "Claude Desktop connectors and extensions managed
   in the claude.ai UI are not stored in this file and are not visible to mcp-medic."
4. Strict-JSON client: if `isStrictJson(text) === false` but JSONC parse succeeds,
   record a `structuralIssues` entry (`kind: 'jsonc-in-strict-client'`) — MM102
   (issue 20) converts it (Claude Desktop itself would reject the file).

# Acceptance Criteria

- [ ] Contract harness passes for all listed fixture cases.
- [ ] `valid-full` fixture contains ≥3 servers incl. one with env secrets and one
      whose name contains `/` (pointer-escaping exercised end-to-end).
- [ ] `wrong-typed-fields` fixture (string `args`) yields a `structuralIssues` entry
      with pointer `/mcpServers/<name>/args` and a correct line number, and the
      entry is still emitted with `args: []`.
- [ ] `jsonc-only` fixture (trailing comma) records `jsonc-in-strict-client`.
- [ ] The adapter never throws on any fixture (harness-enforced).

# Validation

`npm test -- --run tests/unit/discovery/claude-desktop.test.ts`; reviewer cross-checks
paths/fields against `docs/research/client-config-formats.md` §1.

# Dependencies

08.

# Non-goals

- No log-file reading (`~/Library/Logs/Claude` — v2), no `.mcpb`/extension parsing,
  no rules beyond adapter-emitted MM102 inputs.

# Design References

- DESIGN.md §7.2; research doc §1 (Claude Desktop)
