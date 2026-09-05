# Title

Adapter: Claude Code — `~/.claude.json` user & local scopes

# Summary

Implement the first half of the Claude Code adapter: parse `~/.claude.json` top-level
`mcpServers` (user scope) and `projects.<abs path>.mcpServers` (local scope), including
the shared claude-code helpers (entry normalization, `${VAR}` expansion detection,
reserved-name data) reused by issue 11.

# Context

Facts: `docs/research/client-config-formats.md` §2. `~/.claude.json` is a large mixed
state file (123 project entries observed locally) where MCP data is a small subset —
the parser must ignore unrelated keys, tolerate huge files, and treat missing keys as
healthy. Claude Code entries support `type` (stdio/http/sse/ws + `streamable-http`
alias), `url` без `type` being a real client-side error (MM103), and `${VAR}`/`${VAR:-def}`
expansion whose unresolved references break the client (MM105).

# Scope

- `src/discovery/adapters/claude-code/shared.ts` (entry normalizer, expansion scanner,
  `RESERVED_NAMES` const, settings-file readers used by issue 11)
- `src/discovery/adapters/claude-code/user-local.ts` (+ registration)
- Fixtures `tests/fixtures/homes/claude-code-user-local/*` (standard five + quirk cases below)
- `tests/unit/discovery/claude-code-user-local.test.ts`

# Detailed Requirements

1. Candidate: `<home()>/.claude.json`, format `json`, one candidate serving two scopes
   (the adapter emits entries with `scope: 'user'` and `scope: 'local'` from one file).
   Note: 5 MiB cap from issue 06 applies; real files can be large — if over cap, MM107
   (documented limitation).
2. Shared entry normalizer
   `normalizeClaudeEntry(name: string, raw: unknown, source: SourceLocation, ctx: DiscoveryContext)`
   (env expansion reads `ctx.env`; tests inject fake env through it; issue 11 calls
   this exact signature):
   - This adapter records facts only — it emits no MM103/MM105/MM503/MM505 findings
     itself; issues 20/24 own those rules (DESIGN §7.1 principle).
   - Fields: `type`, `command`, `args`, `env`, `url`, `headers`, `headersHelper`,
     `timeout`, `alwaysLoad`, `oauth` — unknown fields preserved in `clientSpecific`.
     Wrong-typed containers/fields (non-object `mcpServers`/`projects`/entry;
     non-array `args`; non-object `env`/`headers`) follow the same salvage rules as
     issue 09 and append `structuralIssues` entries with exact pointers (MM102
     input).
   - `type: 'streamable-http'` → transport `http`; unrecognized declared `type`
     (e.g. `'bogus'`) → transport `unknown` + `clientSpecific.declaredType` recorded
     (MM104 input).
   - `url` present && `type` absent → transport `unknown` + mark
     `clientSpecific.urlWithoutType = true` (MM103 input).
   - Expansion scan: regex `\$\{([A-Z0-9_]+)(:-[^}]*)?\}` over `command`, each `args`
     element, `env` values, `url`, `headers` values. For each match without `:-`
     default and without `ctx.env[NAME]` defined → push name into
     `clientSpecific.unresolvedVars` (MM105 input). Do **not** perform substitution —
     entries keep raw strings (probes for entries with unresolvedVars are ineligible).
   - `RESERVED_NAMES = ['workspace', 'claude-in-chrome', 'computer-use', 'Claude Preview', 'Claude Browser']`
     exported (MM503 input; comparison exact, case-sensitive).
3. User scope: top-level `mcpServers` object → entries `scope:'user'`,
   pointer `/mcpServers/<name>`.
4. Local scope: for each key P of top-level `projects` object where
   `projects[P].mcpServers` is a non-empty object → entries `scope:'local'`, pointer
   `/projects/<escaped P>/mcpServers/<escaped name>` (full RFC 6901 escaping, in
   order: `~` → `~0`, then `/` → `~1` — project paths always contain `/` and may
   contain `~`).
   Record `clientSpecific.projectPath = P` and
   `clientSpecific.projectPathExists = fs.existsSync(P)` (MM505 input).
   Emit local-scope entries for **all** projects, not just `ctx.projectDir` (the
   duplicate/stale rules need the full picture); mark
   `clientSpecific.activeForProject = (P === ctx.projectDir)`.
5. `enabled`: `true` for both scopes (user/local entries have no disable list).
6. Quirk fixtures beyond the standard five: `url-no-type`, `unknown-type`
   (`type: 'bogus'` → `declaredType` recorded, transport `unknown`, no
   `urlWithoutType` flag), `unresolved-var`, `reserved-name`, `stale-project`
   (projects path that does not exist), `streamable-http-alias`,
   `wrong-typed-containers`.

# Acceptance Criteria

- [ ] Contract harness green on standard cases; quirk fixtures produce exactly the
      `clientSpecific` flags listed above (asserted field-by-field).
- [ ] A 123-project fixture with 2 projects having `mcpServers` yields entries only
      for those 2, with correct pointers and `projectPathExists` values.
- [ ] `${TOKEN}` with `TOKEN` set in ctx.env → no unresolvedVars; `${TOKEN:-x}` unset →
      none; `${TOKEN}` unset → `['TOKEN']`.
- [ ] `ws` type entries normalize to transport `ws`.

# Validation

`npm test -- --run tests/unit/discovery/claude-code-user-local.test.ts`; reviewer
cross-checks against research doc §2 (scopes table, fields, expansion, reserved names).

# Dependencies

08.

# Non-goals

- `.mcp.json` project scope, settings enable/disable state, managed scope (issue 11);
  claude.ai connectors (invisible; note emitted in issue 11); rules themselves.

# Design References

- DESIGN.md §7.2 (claude-code row), §10 MM103/MM105/MM503/MM505; research doc §2
