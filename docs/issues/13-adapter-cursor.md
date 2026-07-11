# Title

Adapter: Cursor

# Summary

Implement `src/discovery/adapters/cursor.ts`: parse `~/.cursor/mcp.json` (global) and
`<project>/.cursor/mcp.json` (project) with stdio and remote entries, including the
`auth` OAuth-credential object and `envFile` support.

# Context

Facts: research doc §4. Cursor uses `mcpServers`; stdio entries document
`type: "stdio"` as required plus `command`/`args`/`env`/`envFile`; remote entries use
`url`/`headers`/`auth` (`CLIENT_ID`, `CLIENT_SECRET`, `scopes`). `CLIENT_SECRET` is
literal secret material in a config file (MM401 input via the standard secret scan on
`clientSpecific`? No — see requirement 3: map into a scanned field).

# Scope

- `src/discovery/adapters/cursor.ts` (+ registration)
- Fixtures `tests/fixtures/homes/cursor/*`
- `tests/unit/discovery/cursor.test.ts`

# Detailed Requirements

1. Candidates: `<home()>/.cursor/mcp.json` (scope `user`),
   `<ctx.projectDir>/.cursor/mcp.json` (scope `project`); `format: 'json'` —
   Cursor's documented format is JSON (research doc §4); parsing is still tolerant
   because issue 06 routes all JSON sources through the JSONC reader, and Cursor is
   not on the strict-JSON-client list (no `jsonc-in-strict-client` issue recorded).
2. Entry mapping:
   - stdio: `type` (missing `type` with `command` present is tolerated → transport
     `stdio`; record `clientSpecific.missingDeclaredType = true` since docs call it
     required — info-level input only, no dedicated rule in v1), `command`, `args`,
     `env`, `envFile` → `clientSpecific.envFile` (MM410/MM205 input).
   - remote: `url` present without `type` → `normalizeTransport` (issue 04) returns
     `http` via the cursor/windsurf row of the DESIGN §7.3 table; the adapter's only
     job is to record `clientSpecific.transportAssumed = 'http'` when that row
     applied. Do not reimplement the mapping locally.
   - `headers` → `headers`.
   - `auth` object: copy `CLIENT_ID`/`scopes` into `clientSpecific.auth`
     (never fabricate headers from it), and expose `CLIENT_SECRET` through the
     generic scanned bucket
     `clientSpecific.secretCandidates: Array<{key: string; value: string; pointer: string}>`
     (adapter obligation 6 in DESIGN §7.1) with `key: 'auth.CLIENT_SECRET'` and the
     exact JSON pointer — the security rules (issue 23) and redaction consume
     `secretCandidates` generically.
3. `enabled`: `'unknown'` (UI-managed toggle, state not in the file).
4. Fixtures follow the issue-08 layout: the five standard cases (`missing-file`,
   `valid-empty`, `valid-full`, `malformed`, `secrets-planted`) plus quirks
   `remote-with-auth` (CLIENT_SECRET planted), `envfile-ref`, `stdio-missing-type`.

# Acceptance Criteria

- [ ] Contract harness green on standard cases.
- [ ] `remote-with-auth`: `secretCandidates` contains the CLIENT_SECRET value with a
      pointer ending `/auth/CLIENT_SECRET`; transport is `http` with
      `transportAssumed` recorded.
- [ ] `envfile-ref`: `clientSpecific.envFile` holds the raw path.
- [ ] All entries report `enabled: 'unknown'`.

# Validation

`npm test -- --run tests/unit/discovery/cursor.test.ts`; reviewer cross-checks against
research doc §4, especially the auth-object handling and the documented
transport-assumption deviation.

# Dependencies

08.

# Non-goals

- No OAuth flow simulation, no reading Cursor's UI-toggle state DB, no nested
  `.cursor` directory walking beyond project root.

# Design References

- DESIGN.md §7.2 (cursor row), §7.3 + documented deviation; research doc §4
