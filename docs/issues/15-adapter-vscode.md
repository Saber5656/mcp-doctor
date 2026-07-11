# Title

Adapter: VS Code

# Summary

Implement `src/discovery/adapters/vscode.ts`: parse workspace `.vscode/mcp.json` and
user-profile `mcp.json` (default profile + Insiders variant), with the `servers`
top-level key, `inputs` variable references, and JSONC tolerance.

# Context

Facts: research doc §6. VS Code differs on three axes: top-level key is `servers`
(not `mcpServers`), files are JSONC by design, and secrets are supposed to flow through
`inputs` (`${input:id}`) — an entry using `${input:...}` is *good* hygiene and must not
trigger unresolved-variable or secret findings.

# Scope

- `src/discovery/adapters/vscode.ts` (+ registration)
- Fixtures `tests/fixtures/homes/vscode/*`
- `tests/unit/discovery/vscode.test.ts`

# Detailed Requirements

1. Candidates (format `jsonc`):
   - Workspace: `<ctx.projectDir>/.vscode/mcp.json`, scope `project`.
   - User default profile, scope `user`:
     - darwin: `<appData()>/Code/User/mcp.json`
     - win32: `<appData()>/Code/User/mcp.json`
     - linux: `<xdgConfig()>/Code/User/mcp.json`
   - Insiders variants: same three with `Code - Insiders` directory — always
     returned as candidates (`candidateFiles` performs no filesystem I/O; absent
     files surface normally as `exists: false`). Entries parsed from an Insiders
     file get `clientSpecific.variant = 'insiders'`.
   - Non-default profiles (`User/profiles/<id>/mcp.json`) are **not** scanned (Known
     Unknown U2); `notes()` states this limitation once.
2. Entry mapping from `servers.<name>`:
   - `type` ∈ {`stdio`,`http`} per docs; call `normalizeTransport` (issue 04) and,
     when a declared `type` value is unrecognized, record
     `clientSpecific.declaredType = <raw value>` (transport comes back `unknown`;
     MM104 keys off `declaredType` + `unknown`).
   - stdio: `command`, `args`, `env`, `envFile` (→ `clientSpecific.envFile`).
   - http: `url`, `headers`.
   - `sandboxEnabled` → `clientSpecific`.
   - `${input:xyz}` occurrences anywhere: record ids in `clientSpecific.inputRefs`.
     Exemption contract (shared with issue 23): this adapter runs no env-expansion
     scan (so `${input:…}` can never enter `unresolvedVars`), and MM401 (issue 23)
     skips any value that consists entirely of an input reference
     (`/^\$\{input:[^}]+\}$/`). Regression test here asserts `inputRefs` recorded
     and `unresolvedVars` absent.
   - Top-level `inputs` array: store the declared input ids on each entry as
     `clientSpecific.declaredInputs: string[]`. The per-entry duplication is
     deliberate — it avoids extending `AdapterParseResult` with file-level metadata
     for a single client.
3. `enabled`: `'unknown'` (state stored outside mcp.json).
4. `notes()`: mentions `chat.mcp.discovery.enabled` (servers imported from other
   apps may appear in VS Code without being in these files) and the profiles
   limitation.
5. Quirk fixtures: `jsonc-comments`, `input-refs`, `insiders-variant`,
   `unknown-type`.

# Acceptance Criteria

- [ ] Contract harness green; comments/trailing commas parse cleanly with no
      `parseError` and no `jsonc-in-strict-client` structural issue (VS Code is
      JSONC-native).
- [ ] `input-refs`: `${input:api-key}` in env produces `inputRefs: ['api-key']` and
      no `unresolvedVars` (adapter-level facts only; the MM401 skip itself is
      issue 23's acceptance).
- [ ] Insiders fixture: entries carry `variant: 'insiders'`; when the Insiders file
      is absent its candidate reports `exists: false` without findings.
- [ ] `servers` key name asserted (a `mcpServers`-keyed fixture yields zero entries —
      plus nothing else; it is simply not VS Code's schema).

# Validation

`npm test -- --run tests/unit/discovery/vscode.test.ts`; reviewer cross-checks research
doc §6 (paths per OS, `servers` key, inputs mechanism).

# Dependencies

08.

# Non-goals

- Profile enumeration (U2 → v2), `settings.json` `chat.mcp.*` parsing beyond the note,
  remote-tunnel configurations, VSCodium variants.

# Design References

- DESIGN.md §7.2 (vscode row), §20 U2; research doc §6
