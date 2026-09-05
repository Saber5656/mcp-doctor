# Title

Adapter: Windsurf

# Summary

Implement `src/discovery/adapters/windsurf.ts`: parse
`~/.codeium/windsurf/mcp_config.json` — stdio and remote entries where the remote URL
key is `serverUrl` (accepting `url` as fallback), with Windsurf's own
`${env:VAR}` / `${file:path}` interpolation recorded but never executed.

# Context

Facts: research doc §5. Single user-scope file, `mcpServers` key. Windsurf is the one
client whose URL key differs (`serverUrl`) and whose interpolation syntax
(`${env:VAR_NAME}`, `${file:/path}`) differs from Claude Code's. `${file:...}` targets
must never be read (they may point at secrets; reading them buys nothing).

# Scope

- `src/discovery/adapters/windsurf.ts` (+ registration)
- Fixtures `tests/fixtures/homes/windsurf/*`
- `tests/unit/discovery/windsurf.test.ts`

# Detailed Requirements

1. Candidate: `<home()>/.codeium/windsurf/mcp_config.json`, scope `user`,
   `format: 'json'` (Windsurf's documented format is JSON; parsing is tolerant via
   the shared JSONC reader, and Windsurf is not a strict-JSON-listed client).
   All OSes: the `.codeium` directory is home-relative everywhere.
2. Entry mapping:
   - stdio: `command`, `args`, `env`.
   - remote: `serverUrl` preferred; if only `url` present use it; if **both** present
     use `serverUrl` and append a `structuralIssues` entry
     (`kind: 'duplicate-url-keys'`, pointer to the entry) so MM102 reports it
     (Known Unknown U3).
     Transport: `normalizeTransport` (issue 04) returns `http` via the
     cursor/windsurf row of DESIGN §7.3; record `clientSpecific.transportAssumed`
     when applied (the file cannot distinguish Streamable HTTP from SSE — documented
     in the research doc).
   - `headers` → `headers`.
3. Interpolation scan over `command`, `args[]`, `env` values, the **selected** URL
   field (`serverUrl` or fallback `url` — whichever requirement 2 chose), and
   `headers` values:
   - `${env:NAME}` where `ctx.env.NAME` undefined → `clientSpecific.unresolvedVars`
     (MM105 input; same consumer as Claude Code).
   - `${file:PATH}` → `clientSpecific.fileRefs: string[]` (record PATH; do not read;
     if `expandTilde(PATH)` does not exist → this is MM205 input via
     `clientSpecific.missingFileRefs`).
4. `enabled`: `'unknown'` (UI-managed).
5. Quirk fixtures: `serverurl-remote`, `both-url-keys`, `env-interpolation-unset`,
   `file-interpolation-missing`.

# Acceptance Criteria

- [ ] Contract harness green.
- [ ] `both-url-keys`: entry uses `serverUrl` value, flag recorded.
- [ ] `${env:FOO}` unset → `unresolvedVars: ['FOO']`; set → empty.
- [ ] `${file:/nope}` → `fileRefs` recorded, `missingFileRefs` includes it, and the
      test asserts the file was never opened (spy on fs, or fixture path outside the
      fake home that would throw if read).

# Validation

`npm test -- --run tests/unit/discovery/windsurf.test.ts`; reviewer cross-checks
research doc §5.

# Dependencies

08.

# Non-goals

- No Windsurf plugin-store state, no project scope (none documented), no reading of
  `${file:}` targets under any circumstances.

# Design References

- DESIGN.md §7.2 (windsurf row), §20 U3; research doc §5
