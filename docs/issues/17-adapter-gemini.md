# Title

Adapter: Gemini CLI

# Summary

Implement `src/discovery/adapters/gemini.ts`: parse `mcpServers` from
`~/.gemini/settings.json` (user) and `.gemini/settings.json` (project), map the
three transport-by-key variants (`command`/`url`/`httpUrl`), read the enablement
state file, and surface `trust: true` for the security rule.

# Context

Facts: research doc §8. Gemini distinguishes transport purely by which key is present
(`command` = stdio, `url` = SSE, `httpUrl` = streamable HTTP), supports `cwd`,
`timeout`, `trust`, `includeTools`/`excludeTools`, `$VAR`/`${VAR}` env expansion, and
persists enable/disable in `~/.gemini/mcp-server-enablement.json` (schema = Known
Unknown U8 — tolerant reader).

# Scope

- `src/discovery/adapters/gemini.ts` (+ registration)
- Fixtures `tests/fixtures/homes/gemini/*` (five standard cases + quirks below)
- `tests/unit/discovery/gemini.test.ts`
- `docs/research/client-config-formats.md` §8 update recording the observed
  enablement-file schema (U8 resolution)

# Detailed Requirements

1. Candidates (format `json`): `<home()>/.gemini/settings.json` (scope `user`),
   `<ctx.projectDir>/.gemini/settings.json` (scope `project`).
2. Entry mapping from `mcpServers.<name>`:
   - Exactly one of `command`/`url`/`httpUrl` expected. The adapter selects the
     effective `urlKey` (`httpUrl` if present, else `url`, else none) and calls
     `normalizeTransport({client: 'gemini-cli', hasCommand, urlKey})` (issue 04 owns
     the §7.3 mapping: `httpUrl` → `http`, `url` → `sse`, command-only → `stdio`).
     When more than one transport key is present, additionally set
     `clientSpecific.ambiguousTransport = true` (MM102 input). When **none** is
     present, the entry is still emitted with transport `unknown` (MM106 input).
   - Wrong-typed fields follow issue 09's salvage rules with `structuralIssues`
     kinds `servers-key-not-object`, `entry-not-usable` (entry not an object),
     `field-wrong-type`, each with exact pointers.
   - `args`, `env`, `headers`, `cwd` → fields / `clientSpecific.cwd`.
   - `timeout` → `clientSpecific.timeoutMs`.
   - `trust` (boolean) → `clientSpecific.trust` (MM409 input).
   - `includeTools`/`excludeTools` → `clientSpecific`.
   - Env expansion scan (gemini-specific scanner in this file — do not reuse the
     claude-code one). Grammar: `\$([A-Za-z_][A-Za-z0-9_]*)` (bare form) and
     `\$\{([A-Za-z_][A-Za-z0-9_]*)\}` (braced form) over `command`, `args[]`, `env`
     values. No default-value syntax exists in Gemini docs, so every reference to a
     variable absent from `ctx.env` goes into `unresolvedVars` (MM105 input).
     `$$`, `$1`, and other non-matching sequences are ignored. Output de-duplicated
     and sorted lexicographically (deterministic).
   - `cwd` present but directory missing → `clientSpecific.missingCwd = true`
     (MM205 input variant).
3. Enablement: read `<home()>/.gemini/mcp-server-enablement.json` tolerantly with a
   reader that tries, in order: (a) `{"<name>": bool}`, (b) `{"servers": {"<name>": bool}}`,
   (c) `{"disabled": [names]}` — first shape that matches wins. Semantics:
   shapes (a)/(b): explicitly listed names get their boolean; unlisted names →
   `'unknown'`. Shape (c): listed names → `false`; unlisted → `'unknown'` (a
   disabled-list cannot prove positive enablement). File absent → all `'unknown'`.
   File present but no shape matches → all `'unknown'` and append
   `notes: ['Gemini enablement file present but schema not understood (drift)']` to
   the `AdapterParseResult` (parse-time notes are merged into discovery notes —
   contract extension in issue 08). U8 resolution: record the actually-observed
   schema in the research doc during implementation (research doc is in Scope).
4. Quirk fixtures: `three-transports`, `trust-true`, `enablement-file`,
   `bare-dollar-expansion`, `ambiguous-keys`.

# Acceptance Criteria

- [ ] Contract harness green; `three-transports` yields stdio/sse/http entries
      respectively (asserted per entry).
- [ ] `trust-true` sets `clientSpecific.trust === true`.
- [ ] Enablement matrix: shape (a) fixture — listed true/false honored, unlisted
      `'unknown'`; shape (c) fixture — listed `false`, unlisted `'unknown'`;
      unknown-shape fixture — all `'unknown'` + the drift note present.
- [ ] U8 recorded: research doc §8 names the observed schema with a verification
      date (reviewer checks the diff).
- [ ] `$API_KEY` unset yields `unresolvedVars: ['API_KEY']`; `$HOME` set yields none.

# Validation

`npm test -- --run tests/unit/discovery/gemini.test.ts`; reviewer cross-checks research
doc §8 (field table, transport-by-key, enablement path).

# Dependencies

08.

# Non-goals

- System-level enterprise settings, extension configs, `mcp` global discovery object
  (`serverCommand`/`allowed`/`excluded`) beyond ignoring it silently.

# Design References

- DESIGN.md §7.2 (gemini row), §7.3, §20 U8; research doc §8
