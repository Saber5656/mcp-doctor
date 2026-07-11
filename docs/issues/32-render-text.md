# Title

Terminal text renderer with sanitization enforcement

# Summary

Implement `src/report/render/text.ts`: the human-facing colored terminal report —
findings grouped by client and file, severity colors, summary footer — with
`sanitizeForTerminal` applied to every interpolated untrusted string and full
`NO_COLOR`/`--no-color`/non-TTY handling.

# Context

DESIGN.md §13 (text). This is the default output; it is also the terminal-injection
attack surface (§14.2 case 2) — hence sanitization is enforced here structurally, not
by convention.

# Scope

- `src/report/render/text.ts`
- `tests/unit/report/text.test.ts` (+ golden snapshots under
  `tests/fixtures/render/text/*.txt`)

# Detailed Requirements

1. Layout (top to bottom):
   - Header: `mcp-medic v<version> — scanned <n> servers across <m> clients` +
     mode line (`static only` / `+connect` / `+online`). `<n>` =
     `inventory.servers.length`; `<m>` = count of distinct client ids appearing in
     `inventory.servers` or in `inventory.configFiles` with `exists: true`.
   - Per client (only clients with files found or findings): client display name,
     then per file: relative-to-home path (`~/…`), then findings as
     `  <SEV>  <ruleId>  <title>` + wrapped detail + `fix:` remediation line +
     `at <file>:<line>` when location present.
   - Probe results line per probed server (`ok 812ms · 12 tools` / `timeout` …)
     under its client section when `--connect` ran.
   - Visibility notes section (`note:` prefix).
   - Summary footer: `✖ 2 errors  ⚠ 5 warnings  ℹ 3 info  (1 suppressed)` +
     the failing-threshold sentence when exit will be 1, + hint line
     `run with --connect for live checks` shown iff
     `!report.run.mode.connect && report.inventory.servers.some(s => s.enabled !== false)`
     (a deliberately simple precomputable condition — real probe eligibility is
     issue 28's concern, not the renderer's).
2. Construction rule: every string not authored inside mcp-medic source
   (names, paths, details, remediations containing config values, probe details)
   passes through a single local helper `u(s)` = `sanitizeForTerminal(s)`; authored
   literals are exempt. Enforcement is twofold: code review of every interpolation
   site, plus the adversarial snapshot test in requirement 4 (fixtures with ANSI/OSC
   planted in every untrusted field must render clean) — no runtime instrumentation.
3. Exported API:
   `renderText(report: Report, opts: {color: 'auto' | 'always' | 'never'; isTTY: boolean}): string`
   — no process-global reads inside the renderer. The **CLI** (issue 34) computes
   `color`: `--no-color` or `NO_COLOR` env ⇒ `'never'` (these always win);
   otherwise `'auto'`. `'always'` exists for tests. Effective coloring:
   `never` → off; `always` → on; `auto` → on iff `opts.isTTY`.
   Colors via `picocolors`: error=red, warn=yellow, info=blue, ok=green. Snapshots
   recorded with `color: 'never'`; one test asserts ANSI present with
   `color: 'always'` and absent with `never`/`auto`+non-TTY.
4. Adversarial fixture: a Report **built through `assembleReport`** (secrets are
   masked upstream — the renderer's own guarantee is sanitization, not redaction)
   where server name, file path, finding detail, and probe detail each embed
   `\x1b]0;pwned\x07` and `\x1b[2J`, with fake secrets planted in the pre-assembly
   inventory — rendered output contains none of: `\x1b`, `\x07`, the raw secret
   (pipeline assertion: masking upstream + sanitization here).
5. Width handling: wrap detail/remediation at 100 cols (soft; no wrapping of paths);
   no external wrap dependency (simple word-wrap helper local to the module).
6. Renders to a string; no direct stdout writes (CLI owns streams).

# Acceptance Criteria

- [ ] Golden snapshots for: healthy-empty, static-findings-mix, connect-mode with
      probe lines, suppressed-only, notes-present.
- [ ] Adversarial fixture renders with zero ESC/BEL bytes (byte-level assertion).
- [ ] Color on/off matrix behaves per rule 3.
- [ ] Summary counts match report.summary exactly (property test over random reports
      from a small generator).

# Validation

`npm test -- --run tests/unit/report/text.test.ts`; reviewer eyeballs the golden files
for readability (formatting is a product surface).

# Dependencies

31, 07.

# Non-goals

- No progress/spinner output (CLI, issue 34), no interactive elements, no markdown.

# Design References

- DESIGN.md §13 (text), §12, §14.2 case 2
