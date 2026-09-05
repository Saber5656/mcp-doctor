# Title

CLI wiring: commands, flags, consent gate, orchestrator, exit codes

# Summary

Implement the commander-based CLI (`scan` default, `list`, `clients`, `rules`), the
`--connect` consent gate UI, the orchestrator pipeline that binds discovery → rules →
probe → online → report, stream discipline (stdout report / stderr progress), signal
handling, and the full exit-code contract.

# Context

DESIGN.md §2 (CLI contract, table of flags, exit codes, consent flow), §5
(orchestrator), §15 (failure modes). This is the integration point: every prior module
is consumed here, and the CLI behavior *is* the product contract.

# Scope

- `src/cli/main.ts`, `src/cli/commands/{scan,list,clients,rules}.ts`
- `src/core/orchestrator.ts`
- `src/index.ts` (programmatic `scan(options): Promise<Report>` — same pipeline minus
  process concerns)
- `tests/integration/cli/*.test.ts` (spawning the built CLI against fixture homes)

# Detailed Requirements

1. Commander setup: program name `mcp-medic`, version from package (single source),
   `exitOverride` mapping commander usage errors → exit **2** with a one-line message
   + `run mcp-medic --help`; unknown command → 2.
2. Implement every flag from DESIGN.md §2.2 with exactly the documented names,
   defaults, and repeatability. Validation errors (bad `--client` id, `--fail-on`
   value, `--concurrency` out of 1–16, `--timeout` < 1000) → exit 2 naming the flag.
3. Orchestrator (`runScan(options, io)`) executes DESIGN.md §5 steps 1–7 with the
   documented degradation rules (§15 table): adapter crash → note; registry down →
   note; probe failures → findings. **Two-phase rule evaluation**: phase A runs
   `evaluate()` with static inputs only — its findings feed the probe runner's
   `opts.staticFindings` eligibility filter (issue 28) and the consent preview; when
   neither `--connect` nor `--online` ran, phase A's result is final. Phase B (only
   when probes/online ran) re-runs `evaluate()` with full `EvaluateOptions`
   (`probes`, `online`) and is authoritative — rules are pure, so static findings
   are reproduced identically. `io` injects streams + TTY flags + clock for
   testability.
4. Consent gate (§2.4) exactly:
   - Static phase completes first.
   - Plan preview to **stderr**: one line per probe-eligible server —
     `client:scope:name → <command> <args…>` (argv joined with spaces, each token
     sanitized; env listed as `env: KEY1, KEY2` names only) or `→ GET <masked url>`.
   - TTY + no `--yes`: prompt `Execute these N servers? [y/N] ` on stderr; anything
     but `y`/`Y` → skip probes, continue with static-only report + note.
   - Non-TTY + no `--yes`: exit 2 with
     `--connect requires --yes when not running interactively`.
   - Pass `consentGranted: true` to the runner only on yes/`--yes`.
5. Streams: report → stdout only; everything else (banner, progress from
   `onProgress`, prompts, verbose) → stderr. `--quiet` silences banner/progress/
   verbose **but never the consent preview or prompt** (security-critical output is
   unsuppressible). `--output <file>`: atomic write (tmp + rename), then a one-line
   stderr confirmation; unwritable → exit 2.
6. Signals: SIGINT → `runner.abortAll()`, render the partial report to the normal
   destination (`--output <file>` when given, else stdout) with a
   `partial: interrupted` visibility note + a stderr notice, exit **130**. Second
   SIGINT → immediate exit 130 without report.
7. Exit codes end-to-end: 0/1 via `reportExitCode`; 2 usage (incl. custom-adapter
   `CustomAdapterError`, `PathResolutionError`); 3 unexpected exception (message to
   stderr; stack only with `--verbose`); 130 interrupt.
8. `list`/`clients`/`rules` commands render via the §13-defined renderers/tables and
   always exit 0 (except usage errors). Their `--format json` shapes (exact, all
   with `schemaVersion: 1`):
   - `list`: `{schemaVersion, generatedAt, servers: [<report server projection —
     same shape as Report.inventory.servers>]}`.
   - `clients`: `{schemaVersion, clients: [{id, displayName, detected: boolean,
     files: [{path, exists, scope}]}]}`.
   - `rules`: `{schemaVersion, rules: <ruleTable() rows>}`.
9. `src/index.ts` exports `scan(options)` returning the `Report` (no process.exit, no
   streams) — thin wrapper over the orchestrator; documents that programmatic use
   never prompts (consent must be passed as `connect: {consent: true}`). Also
   re-exports `ruleTable` (issue 19) and the core model types — the docs generator
   (issue 36) imports `ruleTable` from the built `dist/index.js`.

# Acceptance Criteria

- [ ] CLI integration matrix against fixture homes: default scan (no findings → 0;
      planted error → 1 with `--fail-on error`; warn-only + `--fail-on warn` → 1;
      `--fail-on never` → 0).
- [ ] `--connect` non-TTY without `--yes` → exit 2, no child processes spawned
      (marker-file assertion).
- [ ] `--connect --yes` against a fixture home configured with `ok.mjs`/`crash.mjs`
      probes both and reports MM604 for the crasher.
- [ ] Consent preview lists exact argv and masks env values (snapshot).
- [ ] stdout contains only the report for `--format json` (stderr has the rest);
      `mcp-medic scan --format json | jq .` succeeds in test.
- [ ] SIGINT during probes → exit 130, partial JSON parses, no orphans.
- [ ] `--ignore MM999` (unknown) → exit 2; `--ignore MM404` suppresses and exit
      reflects remaining findings.
- [ ] Version/`--help` snapshots stable.

# Validation

`npm test -- --run tests/integration/cli` on the CI matrix; manual smoke on a real
machine: `node dist/cli.js clients` + `scan --format json` (reviewer-run, output
attached to PR).

# Dependencies

08, 18, 19, 28, 29, 31, 32, 33 (adapters 09–17 required for real-machine usefulness
but the CLI compiles/tests against whichever are registered).

# Non-goals

- No config file for defaults (v2), no `--only`, no shell completions, no update
  checks.

# Design References

- DESIGN.md §2 (entire), §5, §15; ADR-003 (consent), ADR-004 (offline default)
