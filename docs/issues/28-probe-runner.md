# Title

Probe runner: eligibility, dedupe, concurrency, and MM605/MM610

# Summary

Implement `src/probe/runner.ts`: select probe-eligible entries, deduplicate identical
servers, run stdio/HTTP probes under a bounded pool with a global phase budget,
aggregate tool lists, and emit MM605 (tools missing/empty) and MM610 (tool-name
collision) — plus Ctrl-C cleanup.

# Context

DESIGN.md §11.1 (eligibility), §11.4 (concurrency/budget), §15 (Ctrl-C). The runner is
the only component that calls `probeStdio`/`probeHttp`; the consent gate itself lives
in the CLI (issue 34) — the runner *requires* an explicit `consentGranted: true`
option and refuses to run otherwise (defense in depth).

# Scope

- `src/probe/runner.ts` —
  `createProbeRunner(inventory, opts): { run(): Promise<{probes: ProbeResult[]}>; abortAll(): void }`
  (findings are not produced here — the engine converts ProbeResults via probe
  rules; the handle shape lets the CLI call `abortAll()` while `run()` is pending)
- `src/rules/probe/aggregate-rules.ts` — `Rule` objects (appliesTo `probe`) MM605
  and MM610 (need the full ProbeResult set, hence "aggregate")
- `tests/integration/probe/runner.test.ts` + `tests/unit/rules/probe/aggregate-rules.test.ts`

# Detailed Requirements

1. `opts.consentGranted !== true` → throw `ConsentError` (the CLI never lets this
   happen; the throw is the safety net and is tested).
2. Eligibility filter (in order, each exclusion recorded as
   `ProbeResult{status:'skipped', detail:<reason>}`):
   - `enabled === false` → skipped (`disabled`).
   - transport `ws` → skipped (`ws not probed in v1`).
   - transport `unknown` → skipped (`transport unknown`).
   - `clientSpecific.inventoryOnly` (zed extension-managed / unknown source) →
     skipped (`inventory-only`).
   - `clientSpecific.unresolvedVars` non-empty → skipped (`unresolved variables` —
     this is the MM105-equivalent input; DESIGN §11.1's MM105 exclusion is
     implemented through this recorded fact).
   - Entries already carrying error-severity static findings from
     MM101/MM102/MM106/MM201/MM204/MM206 (passed in via `opts.staticFindings`) →
     skipped (`static errors present`).
3. Dedupe: group by `serverIdentity` (issue 22); probe one representative per
   identity; clone the `ProbeResult` to every member entry (each clone keeps its own
   `serverKey`). Null identity → probe individually.
4. Pool: `opts.concurrency` (1–16, default 4). Per-probe timeout from §11.2/§11.3.
   Global phase budget = `timeoutMs × ceil(n/concurrency) + 10_000` ms; on breach,
   cancel outstanding probes (abort + kill) and mark them
   `skipped`/`phase budget exceeded`.
5. Aggregate probe rules (Rule objects, appliesTo `probe`, evaluated by the engine
   over the full `opts.probes` array):
   - **MM605** (warn): probe ok AND (tools/list failed OR (`toolCount === 0` AND
     `toolsCapability === true`)).
   - **MM610** (warn): within each client — entries of one client whose
     `enabled !== false` — collect `toolNames` across probed servers; any tool name
     exposed by ≥2 servers → one finding per (client, toolName) listing the server
     names.
6. Progress callbacks `opts.onProgress(evt)` (`start`, `done`, `skipped` per server)
   — consumed by the CLI for stderr progress; no console I/O inside the runner.
7. SIGINT handling: `abortAll()` triggers each in-flight probe's `AbortSignal`
   (issues 26/27 kill/cleanup semantics), marks pending probes `skipped`, and lets
   `run()` resolve normally (never rejects) — enabling the CLI's partial report +
   exit 130 behavior (wiring to the process signal happens in issue 34).

# Acceptance Criteria

- [ ] Eligibility matrix test: one entry per exclusion reason → exact skip reasons.
- [ ] Dedupe: same fixture command configured in 3 fake clients → exactly 1 child
      process observed (fixture writes a spawn-marker temp file; count == 1), 3
      ProbeResults with distinct serverKeys.
- [ ] Concurrency: 8 slow fixtures, concurrency 4 → peak concurrent children ≤ 4
      (marker-file counting), total time ≈ 2 batches not 8.
- [ ] Budget breach: 3 hang fixtures, concurrency 1, timeout 1 s → phase ends within
      budget, remaining marked skipped.
- [ ] MM605: `no-tools.mjs` fixture (tools capability, zero tools) → MM605 fires;
      ok fixture with tools → silent; tools/list-failure synthetic ProbeResult →
      fires.
- [ ] MM610: two ok-fixture servers with overlapping tool name (via `OK_TOOL_PREFIX`)
      in one client fire; same overlap across *different* clients does not.
- [ ] `consentGranted` absent → `ConsentError` thrown before any spawn (marker count 0).
- [ ] `abortAll` mid-run: no orphans, promise resolves, partial results present.

# Validation

`npm test -- --run tests/integration/probe/runner.test.ts` on CI matrix.

# Dependencies

19, 22 (`serverIdentity` for dedupe), 26, 27.

# Non-goals

- No TTY prompting/consent UI (34), no report assembly (31), no online checks.

# Design References

- DESIGN.md §11.1, §11.4, §15 (Ctrl-C row), §10 MM605/MM610; §2.4 (gate)
