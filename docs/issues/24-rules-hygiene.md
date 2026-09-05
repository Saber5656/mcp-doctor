# Title

Hygiene rules MM501–MM505

# Summary

Implement the five hygiene rules: disabled-but-configured servers, deprecated SSE
transport, Claude Code reserved names, non-portable server names, and stale
local-scope project entries.

# Context

DESIGN.md §10 (Hygiene table). Inputs already recorded by adapters:
`enabled:false`/lists (12, 11, 17), transport `sse` (any), `RESERVED_NAMES` (10),
`projectPathExists` (10).

# Scope

- `src/rules/hygiene/mm501.ts` … `mm505.ts` (+ barrel)
- `tests/unit/rules/hygiene/*.test.ts`

# Detailed Requirements

1. **MM501** (entry, info): `enabled === false`. Detail: definition retained and may
   hold stale secrets/paths; remediation: remove the entry if permanently unused.
   Client-specific removal commands where documented: `claude mcp remove <name>`,
   `codex mcp remove <name>`; for gemini-cli (no documented remove command in the
   research doc): "delete the `mcpServers.<name>` block from the settings file";
   generic fallback for other clients: "delete the entry at <pointer> in <file>".
2. **MM502** (entry, warn): `transport === 'sse'`. Under the §7.3 normalization
   table only two clients can produce `sse`: **claude-code** (`type: "sse"`) and
   **gemini-cli** (`url` key). Detail: SSE transport is deprecated in the MCP spec
   (research doc: protocol facts). Remediation table (two rows only):
   claude-code → `change "type" to "http" (or "streamable-http") and point "url" at
   the server's streamable HTTP endpoint`; gemini-cli → `move the endpoint from
   "url" to "httpUrl"`. Any other client id with transport `sse` is impossible by
   construction — assert via a negative test, not runtime code.
3. **MM503** (entry, error, clients: ['claude-code']): `name ∈ RESERVED_NAMES`
   (import from adapter shared module — single source). Detail: Claude Code skips the
   entry at load time.
4. **MM504** (entry, info): `name` matches `/[^A-Za-z0-9_-]/`. Detail: breaks
   `claude mcp add-from-claude-desktop` import and cross-client portability;
   remediation: rename using letters/digits/hyphen/underscore.
5. **MM505** (entry, info, clients: ['claude-code']):
   `clientSpecific.projectPathExists === false`. Detail: local-scope servers defined
   for a project directory that no longer exists; remediation:
   `claude mcp remove <name>` from that project context or edit `~/.claude.json`
   `projects.<path>` (exact pointer included).

# Acceptance Criteria

- [ ] Each rule ≥1 firing + ≥1 non-firing test.
- [ ] MM503 uses the shared constant (test imports both and asserts identity — no
      copy-paste list).
- [ ] MM502 remediation text: table-driven test over claude-code and gemini-cli
      (the only clients that can normalize to `sse` per DESIGN §7.3); a negative
      test documents that no other client id reaches the rule.
- [ ] MM504 boundary: name with space fires; `A-z_0-9-` silent; unicode fires.

# Validation

`npm test -- --run tests/unit/rules/hygiene`; registry snapshot includes all five.

# Dependencies

19; 10 (MM503 imports `RESERVED_NAMES` from
`src/discovery/adapters/claude-code/shared.ts` — single source, no copy).
Issues 11/12/17 provide recorded inputs at runtime; tests use synthetic entries.

# Non-goals

- No probe-based hygiene (MM6xx), no auto-rename/fix.

# Design References

- DESIGN.md §10 Hygiene (MM5xx); research doc §2 (reserved names, import restriction)
