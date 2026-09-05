# Title

Integrity rules MM101–MM108

# Summary

Implement the eight config-integrity rules: parse failures, structural violations,
`url`-without-`type`, unknown transport, unresolved env expansion, incomplete entries,
unreadable/oversized files, duplicate in-file names.

# Context

DESIGN.md §10 (Integrity table) is normative. Most inputs are already recorded by
parsers (issue 06: parse errors, duplicate keys, strict-JSON flag) and adapters
(issues 09–17: `urlWithoutType`, `unresolvedVars`, `ambiguousTransport`) — these rules
convert recorded facts into `Finding`s with exact locations and remediations.

# Scope

- `src/rules/integrity/mm101.ts` … `mm108.ts` (+ barrel registering all)
- `tests/unit/rules/integrity/*.test.ts` (synthetic inventories; reuse adapter
  fixtures where convenient)

# Detailed Requirements

Common: every Finding's `remediation` must name the exact file and, where applicable,
the exact JSON pointer / TOML key path, and be copy-pasteable when a CLI exists for
the fix (e.g. `claude mcp remove <name> -s user`). Adapters emit no findings — these
rules convert recorded data: `ConfigFileInfo.parseError` (MM101),
`ConfigFileInfo.structuralIssues` (MM102), `ConfigFileInfo.loadError` (MM107), and
`clientSpecific` flags (MM103–MM106).

1. **MM101** (file, error): fires when `ConfigFileInfo.parseError` present. Detail =
   raw parser message + line/col; for `client === 'codex'` **this rule** appends
   "Codex cannot start sessions while this file is invalid." (adapters store the raw
   parser message unmodified — wording is owned here).
2. **MM102** (file, error): a single file-scoped rule that (a) converts
   `ConfigFileInfo.structuralIssues` (kinds: `servers-key-not-object`,
   `field-wrong-type`, `entry-not-usable`, `jsonc-in-strict-client`,
   `duplicate-url-keys`, adapter-specific kinds) and (b) scans
   `inventory.servers` for entries with `entry.source.file === file.path` carrying
   `clientSpecific.ambiguousTransport` — one finding per issue, detail names the
   construct, location from the recorded pointer/line.
3. **MM103** (entry, error, clients: ['claude-code']): `clientSpecific.urlWithoutType`.
   Remediation: `Add "type": "http" (or "sse"/"ws") to <file> at <pointer>` —
   mirror Claude Code's own error message from the research doc.
4. **MM104** (entry, error): transport `unknown` AND a `type`-ish raw value was
   present but unrecognized (adapters record `clientSpecific.declaredType`).
   Do not fire when MM103 already fired for the entry (engine de-dup won't catch
   cross-rule overlap — explicit guard).
5. **MM105** (entry, error): `clientSpecific.unresolvedVars` non-empty. Detail names
   each variable and the client's expansion syntax; remediation: export the variable,
   add a default (`${VAR:-default}` where the client supports it), or use the
   client's secret mechanism.
6. **MM106** (entry, error): no `command` and no url-ish field (transport `unknown`
   with neither). Skip when MM103 fired (that url exists) and skip entries with
   `clientSpecific.inventoryOnly` (Zed extension-managed artifacts are not user
   misconfigurations — issue 16).
7. **MM107** (file, warn): converts `ConfigFileInfo.loadError`
   (kinds `unreadable` / `too-large` / `io-error`). Detail includes errno or size;
   states that coverage is incomplete. Managed-scope claude-code candidates are
   exempt (issue 11 excludes them from the mapping).
8. **MM108** (file, warn): converts `ConfigFileInfo.duplicateKeys` (populated
   generically by discovery, issue 08 — duplicate keys anywhere in a client config
   file are suspect, not only under the servers container). Detail: "last definition
   wins silently in JSON — remove one". TOML duplicate tables surface as MM101
   (parse error), not MM108 — add a test asserting no double-report.

# Acceptance Criteria

- [ ] Each rule has ≥1 firing and ≥1 non-firing test; boundary tests: MM104-vs-MM103
      exclusivity, MM106-vs-MM103 exclusivity, MM108-vs-MM101 TOML case.
- [ ] Every emitted Finding carries `location.file`; line numbers present for all
      JSON-based fixtures.
- [ ] Remediation strings reviewed: file path + pointer + concrete action present in
      100% of findings (test asserts non-empty and contains the file path).
- [ ] Registry lists all eight with correct category/severity (`ruleTable` snapshot).

# Validation

`npm test -- --run tests/unit/rules/integrity`; reviewer diffs behavior line-by-line
against the DESIGN.md §10 Integrity table.

# Dependencies

19, 06, 08 (type contracts: `ConfigFileInfo.structuralIssues`/`duplicateKeys`
population). Adapters 09–17 are **not** dependencies — all tests use synthetic
`Inventory`/`ConfigFileInfo` objects built from the 04/08 types.

# Non-goals

- No PATH/executability checks (21), no security semantics (23), no probe (26–28).

# Design References

- DESIGN.md §10 Integrity (MM1xx); research doc §2 (claude-code error wording)
