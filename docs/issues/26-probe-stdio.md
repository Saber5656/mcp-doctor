# Title

stdio probe engine with hardened spawn and failure classification

# Summary

Implement `src/probe/stdio.ts` + `src/probe/classify.ts` (stdio half): spawn one
stdio server under the hardening rules, drive the MCP handshake and `tools/list` via
the official SDK, enforce timeout/kill-escalation, and classify every outcome into
`ProbeResult` + MM601/602/603/604/608/609/611 findings.

# Context

DESIGN.md §11.2 (state machine + hardening list) and §14.2 case 1. This is the
security-critical execution path: it runs user-configured commands. Every hardening
requirement is testable and tested.

# Scope

- `src/probe/stdio.ts` — `probeStdio(entry, opts): Promise<ProbeResult>` (spawn,
  handshake, classification into `status` + `facts` per DESIGN §4)
- `src/rules/probe/stdio-rules.ts` — `Rule` objects (appliesTo `probe`) MM601,
  MM602, MM603, MM604, MM608, MM609, MM611 converting `ProbeResult` facts into
  findings (MM603 is transport-agnostic: it also fires for HTTP handshake errors
  produced by issue 27)
- `tests/integration/probe/stdio.test.ts` (against issue 25 fixtures) +
  `tests/unit/rules/probe/stdio-rules.test.ts` (synthetic ProbeResults)

# Detailed Requirements

1. Spawn (all mandatory, asserted by tests):
   - `child_process.spawn(entry.command, entry.args, {shell: false, ...})` — never a
     string command, never `shell: true`.
   - Env allowlist base: `PATH`, `HOME`, `USERPROFILE`, `TMPDIR`, `TEMP`, `TMP`,
     `SystemRoot`, `SYSTEMROOT`, `ComSpec`, `LANG`, `LC_ALL` — from `process.env`,
     merged with `entry.env` (entry wins). Nothing else from the operator's
     environment leaks in.
   - POSIX: `detached: true` then kill via `process.kill(-pid, sig)`;
     win32: `taskkill /pid <pid> /T /F` fallback after graceful close fails.
   - `cwd`: `clientSpecific.cwd` when it is a string pointing at an existing
     directory (gemini), else `opts.homeDir` (not the mcp-medic cwd — servers must
     not observe the diagnostic tool's context).
   - stderr piped into a 64 KiB ring buffer (`Buffer` window, older bytes dropped).
2. Handshake: SDK `Client` (`name: 'mcp-medic'`, tool version) over a **custom
   `HardenedStdioTransport`** implementing the SDK `Transport` interface
   (`start`/`send`/`close`/`onmessage`/`onerror`/`onclose`) around the child we
   spawn ourselves per requirement 1. Do **not** use the SDK's
   `StdioClientTransport` (it spawns internally via cross-spawn, which can route
   through a shell on Windows — untestable hardening). Owning the stdout parser is
   also what makes `facts.stdoutNoise` observable: any non-JSON line outside frames
   sets the flag while parsing continues on subsequent valid lines.
   Options type (exported):
   ```ts
   interface ProbeStdioOptions {
     timeoutMs: number;            // default 15000 (runner passes effective value)
     slowThresholdMs: number;      // default 5000
     homeDir: string;              // from paths.home(); used as default cwd
     signal?: AbortSignal;         // runner abort → immediate kill-escalation
   }
   ```
   `probeStdio` returns `Promise<ProbeResult>`; on `signal` abort it kills the child
   (same escalation as shutdown) and resolves with status `skipped`,
   detail `aborted`.
3. Timeline (single deadline `opts.timeoutMs`, default 15000, overridden by
   `min(opts.timeoutMs, clientSpecific.startupTimeoutSec*1000)` when codex set it):
   `t0 spawn → initialize → tools/list → close`. `startupMs` = time to initialize
   response.
4. Shutdown: `transport.close()` → wait ≤ 2 s → SIGTERM (group) → wait ≤ 2 s →
   SIGKILL (group). Always runs (finally); test asserts no orphan by polling
   `process.kill(pid, 0)` post-probe.
5. Outcome → `ProbeResult` (`status` + `facts`), then rules convert:
   | Observation | status + facts | Rule → Finding |
   |---|---|---|
   | spawn error ENOENT/EACCES | `spawn-failed`, `facts.errno` | MM601 (errno, resolved path) |
   | deadline hit pre-initialize | `timeout` | MM602 |
   | initialize JSON-RPC error / invalid result / version negotiation failure | `handshake-error` | MM603 |
   | exit before deadline without handshake, or post-handshake crash | `exited`, `facts.exitCode`/`signal`/`stderrTail` (last 10 lines, redacted+sanitized) | MM604 |
   | non-JSON bytes observed on stdout | final status per outcome + `facts.stdoutNoise: true` | MM608 (fires whenever the flag is set) |
   | handshake ok, `startupMs > opts.slowThresholdMs` (default 5000) | `ok` | MM609 |
   | negotiated protocolVersion < 2025-11-25 | `ok`, `protocolVersion` | MM611 (info) |
   | all good | `ok` | — |
   `tools/list` failure with ok handshake → status `ok`, `toolCount` undefined,
   `toolsCapability` recorded — MM605 (issue 28's aggregate rules) owns that
   conversion. This module still returns `toolNames` on success.
6. All stderr/stdout excerpts pass `redact`+`sanitizeForTerminal` (issue 07) before
   entering `ProbeResult.detail`/findings.
7. `probeStdio` never throws — every internal error becomes a classified outcome
   (`exited`/`spawn-failed` with detail) so one bad server cannot break the run.

# Acceptance Criteria

- [ ] Fixture matrix: ok→ok/no findings; slow(6s, timeout 15s)→MM609;
      slow(6s, timeout 3s)→MM602; hang→MM602 with no orphan process;
      crash pre/post-init→MM604 with exit code and stderr tail;
      garbage-stdout→MM608; stderr-noise→detail contains `[REDACTED` and zero raw
      planted-secret substring, zero ESC bytes.
- [ ] Env-leak test: spawn `ok.mjs` variant printing `process.env` keys to a tool
      response; assert an operator-set canary var (`MCP_MEDIC_CANARY=x` in test env)
      is absent, while `entry.env` values are present.
- [ ] Kill-escalation test using `hang.mjs` with `HANG_IGNORE_SIGTERM=1`
      (knob specified in issue 25) — the process is gone within timeout + 5 s,
      proving the SIGTERM→SIGKILL escalation.
- [ ] No `shell: true` anywhere (static assertion: grep in test over built output is
      acceptable, or spy on spawn options).

# Validation

`npm test -- --run tests/integration/probe/stdio.test.ts` across the CI matrix
(win32 taskkill path included).

# Dependencies

04, 05 (`paths.home()` for `homeDir`), 07 (redaction/sanitization of
stderr/detail), 19 (rule registration for the MM6xx rule objects), 25 (fixtures).

# Non-goals

- No eligibility/concurrency/consent (28), no HTTP (27), no MM605/MM610 emission.

# Design References

- DESIGN.md §11.1–11.2, §14.2 case 1, §12; §10 MM6xx table
