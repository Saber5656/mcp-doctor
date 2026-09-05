# Title

E2E golden tests + security regression suite

# Summary

Build the system-level acceptance suite defined in DESIGN.md §17.3: complete fake-home
environments with planted defects for every rule family, golden text/JSON outputs, the
`all-healthy` zero-findings home, and the product-wide security guarantees (no secret
leakage, no network in default mode, consent gate, orphan-free probes).

# Context

This suite is the v1 definition of done (ISSUE_PLAN "whole-product validation") and
the regression net for every future change. It runs the **built** CLI as a subprocess
— no test doubles.

# Scope

- `tests/fixtures/homes/e2e-full/` — one fake home wiring all 8 clients with planted
  defects (built by composing the per-adapter fixtures). Manifest `planted.json`
  schema: array of
  `{rule: string, mode: 'static' | 'connect' | 'online', count?: number,
    fileSuffix?: string, serverKeySuffix?: string, detailIncludes?: string,
    platforms?: Array<'darwin'|'linux'|'win32'>}`
  — assertions match findings by rule + suffix/predicate; `platforms` scopes
  OS-conditional rules (MM402/MM410 assert on POSIX and assert **absence** on
  win32)
- `tests/fixtures/homes/e2e-healthy/` — realistic, all-valid configs for all clients
- `tests/e2e/*.test.ts` + golden files
- CI wiring: e2e job stage after build (extend issue 02 workflow)

# Detailed Requirements

1. `e2e-full` planting coverage: at least one planted instance for **every rule id**
   in the §10 catalog. MM1xx–MM5xx are planted directly in the fixture configs;
   MM6xx via fixture-server commands (issue 25) wired into the configs; MM7xx via the
   programmatic API's `registryBaseUrl` seam pointed at a local mock registry
   (DESIGN §17 test-only seams — no hidden CLI/env flags in production builds).
   Coverage split, stated in the suite README: CLI-subprocess e2e covers MM1xx–MM6xx;
   programmatic `scan()` e2e covers MM7xx.
2. Assertions over `scan --format json` on `e2e-full`:
   - For every entry in `planted.json`: a finding with that rule id exists at the
     expected file (pointer prefix match) — zero false negatives.
   - Global: no finding whose rule id is not in the catalog; counts match summary.
3. `e2e-healthy` → zero findings, exit 0, for static, `--connect --yes` (fixture
   servers all `ok.mjs`), and programmatic online (registry mock all-found) modes —
   zero false positives.
4. Security regression (each its own test):
   - **Leak-free**: every planted secret string (collected list in `planted.json`)
     absent from stdout+stderr bytes of every mode × format combination.
   - **Sanitization**: planted ANSI/OSC sequences absent from text and markdown
     outputs (byte scan).
   - **Offline default**: run a static scan (no `--online`) while the mock registry
     is listening and assert zero requests reach it, combined with issue 29's
     unit-level guarantee of no import-time I/O. Residual risk (raw sockets cannot be
     intercepted portably) is documented in the suite README and accepted.
   - **Consent**: `--connect` non-TTY without `--yes` spawns nothing (marker files).
   - **Orphans**: after `--connect --yes` over hang/crash fixtures, no fixture PIDs
     alive (PID list captured via marker files containing PIDs).
5. Golden outputs: text, JSON, and markdown for `e2e-full` static mode committed and
   byte-compared. Normalization: timestamps/durations via the sanctioned
   `MCP_MEDIC_FAKE_NOW` seam (DESIGN §17); the tool version is normalized by the
   **golden comparator** (regex-replace `v\d+\.\d+\.\d+` → `vX.Y.Z` before
   comparison) — no version seam exists in the product.
6. Performance smoke: static scan of `e2e-full` completes < 5 s in CI (generous;
   catches pathological regressions only).
7. Cross-platform: full suite in the 3-OS CI matrix; path-sensitive goldens
   normalized (`~`-relative display paths make this feasible — verify).

# Acceptance Criteria

- [ ] Rule-coverage meta-test: `planted.json` rule ids ⊇ every MM id in issue 19's
      `ruleTable()` export minus a documented allowlist (e.g. MM107-too-large
      generated at runtime) — the suite fails when a future rule lands without a
      planted fixture.
- [ ] All §17.3 outcomes green on all 3 OSes.
- [ ] Secret-leak scan covers stdout AND stderr, all formats, static+connect modes.
- [ ] Goldens stable across two consecutive CI runs (determinism).

# Validation

CI matrix green including the new e2e stage; reviewer audits `planted.json` against
DESIGN.md §10 row-by-row.

# Dependencies

34, 09–17 (all adapters), 25 (fixtures), 29 (registryBaseUrl seam), 30 (MM703
coverage).

# Non-goals

- No fuzzing (v2 candidate), no load/perf benchmarking beyond the smoke bound, no
  mutation testing.

# Design References

- DESIGN.md §17.3 (acceptance), §17.1 layers 4–5, §12 guarantee, §14.3
