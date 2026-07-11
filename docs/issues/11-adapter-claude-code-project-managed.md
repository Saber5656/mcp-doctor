# Title

Adapter: Claude Code — `.mcp.json` project scope, settings state, managed scope

# Summary

Complete the Claude Code adapter: parse project `.mcp.json`, resolve per-server
enable/disable/approval state from the settings-file chain, and read enterprise
`managed-mcp.json` when present.

# Context

Research doc §2: project scope lives in `.mcp.json` at the project root (approval
required); enablement state is spread over `enabledMcpjsonServers`,
`disabledMcpjsonServers`, `enableAllProjectMcpServers` in `.claude/settings.json`,
`.claude/settings.local.json`, `~/.claude/settings.json`, and managed settings.
Managed `managed-mcp.json` per-OS path is Known Unknown U1 — this issue resolves it
against https://code.claude.com/docs/en/managed-mcp and records the answer in the
research doc before coding.

# Scope

- `src/discovery/adapters/claude-code/project-managed.ts` (+ registration; reuses
  `shared.ts` from issue 10)
- Research-doc update for U1 (managed paths per OS)
- Fixtures `tests/fixtures/homes/claude-code-project/*` incl. `project/` trees
- `tests/unit/discovery/claude-code-project.test.ts`

# Detailed Requirements

1. Candidates:
   - `<ctx.projectDir>/.mcp.json` — scope `project`, format `json`.
   - Managed: per-OS paths per https://code.claude.com/docs/en/managed-mcp —
     macOS `/Library/Application Support/ClaudeCode/managed-mcp.json`,
     Linux `/etc/claude-code/managed-mcp.json`,
     Windows `C:\Program Files\ClaudeCode\managed-mcp.json` — scope `managed`,
     format `json`. (U1: re-verify against that page at implementation time, update
     `docs/research/client-config-formats.md` §2, and cite it in a code comment.)
2. `.mcp.json` parsing: top-level `mcpServers`; entries via `normalizeClaudeEntry`
   (shared) — inherits MM103/MM105 fact recording, pointers `/mcpServers/<name>`.
3. Enablement resolution for project-scope entries (evaluated in this order, first
   hit wins; record the deciding source in `clientSpecific.enabledSource`):
   1. name ∈ `disabledMcpjsonServers` (any settings file) → `enabled: false`
   2. name ∈ `enabledMcpjsonServers` (any) → `enabled: true`
   3. `enableAllProjectMcpServers: true` (any) → `enabled: true`
   4. otherwise → `enabled: 'unknown'` (approval state not readable) + note
      "project servers may be pending approval; run `claude` to review"
   Settings files read (tolerant, missing ⇒ skip): `<projectDir>/.claude/settings.json`,
   `<projectDir>/.claude/settings.local.json`, `<home>/.claude/settings.json`, and
   managed settings (same directories as `managed-mcp.json` above, file
   `managed-settings.json`) — managed entries in the arrays participate in the same
   precedence (a managed `disabledMcpjsonServers` still disables).
4. Managed file (`managed-mcp.json`) parsing: same entry shape via
   `normalizeClaudeEntry(name, raw, source, ctx)`; scope `managed`;
   `enabled: 'unknown'` with `clientSpecific.enabledSource = 'managed-policy-not-evaluated'`
   (Claude Code additionally applies `allowedMcpServers`/`deniedMcpServers` policy
   that v1 does not model — claiming `true` would overstate what we verified);
   unreadable/absent ⇒ silently absent (root-owned paths are often unreadable for the
   scanning user — the managed candidates are exempt from MM107; note this in the
   adapter and exclude them from the LoadError→MM107 mapping).
5. Emit the claude.ai-connectors visibility note (moved here from issue 10 so it
   appears exactly once for the client): "claude.ai connectors are cloud-managed and
   not visible to file scans."
6. Fixtures: standard five for `.mcp.json` + `enabled-by-list`, `disabled-by-list`,
   `enable-all-flag`, `no-approval-state`, `managed-present`.

# Acceptance Criteria

- [ ] Enablement matrix test covers all resolution outcomes with the deciding source
      asserted: disabled-list beats enabled-list; disabled-list beats
      `enableAllProjectMcpServers: true`; conflicting values across settings files
      resolve per the documented order with `enabledSource` naming the winning file.
- [ ] Managed fixture yields `scope:'managed'` entries with `enabled: 'unknown'` and
      `enabledSource: 'managed-policy-not-evaluated'`; absent managed path yields
      nothing and no recorded issue.
- [ ] U1 resolved: research doc updated with verified managed paths + source URL,
      and the adapter's paths match it.
- [ ] Contract harness green for the `.mcp.json` cases.

# Validation

`npm test -- --run tests/unit/discovery/claude-code-project.test.ts`; reviewer checks
the enablement precedence against research doc §2 and the U1 citation.

# Dependencies

08, 10 (shared helpers).

# Non-goals

- Plugin-provided servers (v1 out of scope; DESIGN §1.4), `--mcp-config` CLI-flag
  sources, workspace-trust simulation beyond the documented settings semantics.

# Design References

- DESIGN.md §7.2, §20 U1; research doc §2
