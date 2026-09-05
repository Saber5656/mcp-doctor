# Title

Adapter: Codex CLI (TOML)

# Summary

Implement `src/discovery/adapters/codex.ts`: parse `[mcp_servers.<name>]` tables from
`~/.codex/config.toml` (user) and `<project>/.codex/config.toml` (project), the only
TOML-based client in v1.

# Context

Facts: research doc §3, verified against local `codex-cli 0.141.0`. Codex refuses to
start any session when the TOML is invalid — a parse error here is more severe in
practice than in other clients, and the MM101 finding detail must say so. Codex
supports stdio (`command`/`args`/`env`) and streamable HTTP (`url`,
`bearer_token_env_var`), plus `startup_timeout_sec` (default 10), `tool_timeout_sec`,
`enabled`.

# Scope

- `src/discovery/adapters/codex.ts` (+ registration)
- Fixtures `tests/fixtures/homes/codex/*` (standard five + quirks)
- `tests/unit/discovery/codex.test.ts`

# Detailed Requirements

1. Candidates: `<home()>/.codex/config.toml` (scope `user`),
   `<ctx.projectDir>/.codex/config.toml` (scope `project`). Every `ServerEntry`
   parsed from the project-scope file gets `clientSpecific.trustGated = true`
   (Codex loads that file only for trusted projects — Known Unknown U5: do not model
   trust further, just annotate each entry).
2. Parse via `parseToml` (issue 06). Whole-file parse error → `parseError` carrying
   the raw parser message (no prefixing — MM101 in issue 20 owns the
   "Codex cannot start sessions…" wording).
3. Extract `mcp_servers` table (absent ⇒ healthy zero entries). For each
   `mcp_servers.<name>`:
   - stdio: `command` (string), `args` (string[]), `env` (table of strings).
   - http: `url` (string) → transport `http` (per §7.3: codex `url` ⇒ http);
     `bearer_token_env_var` (string) → `clientSpecific.bearerTokenEnvVar`; also record
     `clientSpecific.bearerTokenEnvSet = ctx.env[value] !== undefined` (a missing
     variable is probe-relevant and a useful MM105-adjacent detail — but do NOT emit
     MM105 for it; it is optional auth, not expansion).
   - `enabled` (boolean, default true) → `enabled`.
   - `startup_timeout_sec` (number, default 10) and `tool_timeout_sec` →
     `clientSpecific` (probe runner uses startup_timeout_sec × 1000 as the per-server
     timeout override when smaller than `--timeout`).
   - Both `command` and `url` present → record
     `clientSpecific.ambiguousTransport = true` (MM102 input), then call
     `normalizeTransport` with `urlKey: 'url'` (url wins deterministically →
     transport `http`); both `command` and `url` stay populated on the entry for
     display. Actual Codex behavior for this misconfiguration is unverified — the
     MM102 finding is the product answer, the transport choice just keeps the
     pipeline deterministic.
   - Structural validation mirrors issue 09's salvage rules with `structuralIssues`
     kinds: `servers-key-not-object` (`mcp_servers` not a table), `entry-not-usable`
     (entry not a table, or neither `command` string nor `url` string),
     `field-wrong-type` (non-array `args`, non-table `env`, non-boolean `enabled`,
     non-number timeouts — offending field dropped to its default, issue recorded
     with pointer).
   - Pointer: RFC 6901 JSON Pointer over the parsed TOML value —
     `/mcp_servers/<escaped name>` (matches the issue-08 harness's universal
     pointer-resolution assertion). TOML dotted key paths are used only as
     `locateTomlKey` input for line numbers.
4. Quirk fixtures: `disabled-server`, `http-with-bearer-env`, `duplicate-table`
   (parse error), `both-command-and-url`, `startup-timeout-custom`.

# Acceptance Criteria

- [ ] Contract harness green; `duplicate-table` fixture yields file `parseError`
      with the raw parser message and a line number.
- [ ] `enabled = false` server yields `enabled: false` (probe-runner exclusion input).
- [ ] `bearer_token_env_var` with the variable set/unset toggles
      `bearerTokenEnvSet` correctly.
- [ ] Transport mapping: `url` ⇒ `http` asserted (never `sse`).

# Validation

`npm test -- --run tests/unit/discovery/codex.test.ts` (the test file invokes
`adapterContractTests` from issue 08 plus the quirk assertions); reviewer
cross-checks fields against research doc §3 and the `codex mcp add --help` capture
therein.

# Dependencies

08.

# Non-goals

- No execution of codex CLI, no reading of `~/.codex/auth.json` or other Codex state,
  no trust-level modeling beyond the annotation.

# Design References

- DESIGN.md §7.2 (codex row), §7.3, §20 U5; research doc §3
