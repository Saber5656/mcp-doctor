# Title

Adapter: Zed

# Summary

Implement `src/discovery/adapters/zed.ts`: parse `context_servers` from Zed's JSONC
`settings.json` (user `~/.config/zed/settings.json`, project `.zed/settings.json`),
accepting both the flat modern shape and the legacy nested `command` object, and
treating extension-provided entries as inventory-only.

# Context

Facts: research doc §7. Zed embeds MCP config inside its general settings file (JSONC,
users comment heavily), uses `context_servers`, `source: "custom" | "extension"`, and
historically nested the command (`"command": {"path": ..., "args": [...]}`). Zed uses
`~/.config` on macOS too (not Application Support) — a classic path-resolution trap.

# Scope

- `src/discovery/adapters/zed.ts` (+ registration)
- Fixtures `tests/fixtures/homes/zed/*`
- `tests/unit/discovery/zed.test.ts`

# Detailed Requirements

1. Candidates (format `jsonc`):
   - user: `<xdgConfig()>/zed/settings.json` on darwin AND linux (use `xdgConfig()`,
     not `appData()` — add an explanatory comment); win32: `<appData()>/Zed/settings.json`.
   - project: `<ctx.projectDir>/.zed/settings.json`, scope `project`.
2. Entry mapping from `context_servers.<name>`:
   - Shape A (modern): `{ "source": "custom", "command": "cmd", "args": [...], "env": {...} }`.
   - Shape B (legacy): `{ "command": { "path": "cmd", "args": [...] }, "env"?: ... }`
     → normalize `command.path` → `command`, `command.args` → `args`; record
     `clientSpecific.legacyShape = true`.
   - `source: "extension"`: create the entry, set
     `clientSpecific.extensionManaged = true` and
     `clientSpecific.inventoryOnly = true`; `settings` object → `clientSpecific`.
   - Any other unrecognized `source` value: `clientSpecific.inventoryOnly = true` +
     `clientSpecific.unknownSource = <value>` (U4 tolerance — do not conflate with
     extensions).
   - `inventoryOnly` entries are excluded from probe eligibility (issue 28) **and**
     from MM106/MM2xx broken-entry rules (issues 20/21) — they are managed artifacts,
     not user misconfigurations.
   - No documented URL key: custom-source entries without a resolvable command →
     transport `unknown` (MM106 input via generic rule; do not invent remote
     support).
   - Structural validation mirrors issue 09's salvage rules with `structuralIssues`
     kinds `servers-key-not-object` (`context_servers` not an object),
     `entry-not-usable`, `field-wrong-type` (wrong-typed `command`/`args`/`env`),
     each with exact pointers.
3. `enabled`: `'unknown'` (UI-managed).
4. The rest of `settings.json` (hundreds of unrelated keys) must be ignored without
   findings; only `context_servers` is inspected.
5. Quirk fixtures: `legacy-command-object`, `extension-entry`, `heavy-comments`
   (comments + trailing commas everywhere), `unrelated-settings-only` (no
   context_servers key → healthy).

# Acceptance Criteria

- [ ] Contract harness green; both shapes normalize to identical
      `command`/`args` projections (asserted side-by-side).
- [ ] Extension entry flagged `extensionManaged` + `inventoryOnly`, still listed in
      inventory; an entry with `source: "future-thing"` gets `inventoryOnly` +
      `unknownSource: "future-thing"` and not `extensionManaged`.
- [ ] macOS fixture resolves under `~/.config/zed/…` (test pins `platform='darwin'`
      and asserts the exact candidate path).
- [ ] `heavy-comments` yields zero parse findings.

# Validation

`npm test -- --run tests/unit/discovery/zed.test.ts`; reviewer cross-checks research
doc §7 (path trap, shapes, `source` semantics).

# Dependencies

08.

# Non-goals

- No Zed extension registry inspection, no remote context-server speculation (U4),
  no parsing of other settings keys.

# Design References

- DESIGN.md §7.2 (zed row), §20 U4; research doc §7
