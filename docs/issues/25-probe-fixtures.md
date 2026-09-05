# Title

Probe test fixtures: scripted fake MCP servers + HTTP fixtures

# Summary

Build the controllable fake-server corpus that the probe engine issues (26–28) and the
E2E suite (35) test against: seven scripted stdio MCP servers with distinct behaviors
and an in-process HTTP fixture with ok/401/404/500/TLS-bad endpoints.

# Context

DESIGN.md §17.1 layer 3 enumerates the fixture behaviors. Building them first (and
independently) lets the stdio and HTTP probe issues run in parallel against a stable
harness. Fixtures use the official SDK's server API where a real server is needed and
raw stdio scripts where protocol-violating behavior must be simulated.

# Scope

- `tests/fixtures/servers/ok.mjs`, `slow.mjs`, `hang.mjs`, `crash.mjs`,
  `garbage-stdout.mjs`, `no-tools.mjs`, `stderr-noise.mjs`
- `tests/harness/http-fixture.ts` (start/stop helper returning base URLs)
- `tests/harness/spawn-fixture.ts` (helper resolving fixture paths + node argv)
- Smoke tests `tests/unit/fixtures/servers.test.ts`

# Detailed Requirements

1. All stdio fixtures are plain `.mjs` (no build step), runnable as
   `node tests/fixtures/servers/<name>.mjs`, importing `@modelcontextprotocol/sdk`
   from the repo's node_modules. Behaviors:
   - `ok.mjs`: SDK `McpServer` over stdio; declares 2 tools (`echo`, `add`) with
     simple schemas; responds normally. Env knob `OK_TOOL_PREFIX` prefixes tool names
     (used by MM610 collision tests to create same/different tool names).
   - `slow.mjs`: waits `SLOW_MS` (default 6000) before starting the SDK server, then
     behaves like ok. Used for MM609 (slow) and MM602 (timeout when SLOW_MS > timeout).
   - `hang.mjs`: reads stdin forever, never writes a byte. MM602. Env knob
     `HANG_IGNORE_SIGTERM=1` installs a SIGTERM handler that ignores the signal
     (used by issue 26's kill-escalation test).
   - `crash.mjs`: exits with code `CRASH_CODE` (default 1) after `CRASH_AFTER_MS`
     (default 100), optionally after writing a valid `initialize` response when
     `CRASH_AFTER_INIT=1`. MM604 both phases.
   - `garbage-stdout.mjs`: prints 3 lines of non-JSON text to stdout, then runs the
     SDK server normally. MM608 (and proves recovery classification: handshake may
     still succeed depending on SDK framing — classifier decides; fixture just does it).
   - `no-tools.mjs`: SDK server advertising tools capability with zero tools. MM605.
   - `stderr-noise.mjs`: like ok, but continuously writes to stderr lines containing a
     planted fake secret (`sk-fixture123456789012345678901234`) and ANSI escape
     sequences. Redaction/sanitization regression input.
2. `http-fixture.ts`:
   `startHttpFixture(): Promise<{urls: {ok, unauthorized, notFound, serverError, slowStart, sseFallback, oldProtocol}, tlsBadUrl, close()}>`
   — all endpoints bound to `127.0.0.1:0` (ephemeral ports, loopback only):
   - `ok`: minimal streamable-HTTP MCP endpoint via SDK server, 2 tools whose
     **names** encode request state for header assertions: `echo`, plus
     `auth-header-seen` listed only when the request carried an `Authorization`
     header (issue 27 asserts header delivery via `tools/list` names — never via
     header values).
   - `unauthorized`: always `401` with `WWW-Authenticate: Bearer`.
   - `notFound`: `404`. `serverError`: `500`. `slowStart`: accepts, first byte after
     `HTTP_SLOW_MS` (settable to values far above probe timeouts for deterministic
     timeout classification).
   - `sseFallback`: responds `405` to streamable-HTTP POSTs but serves a working
     legacy SSE MCP endpoint (issue 27's fallback test).
   - `oldProtocol`: streamable-HTTP endpoint that negotiates protocol version
     `2024-11-05` (issue 27's MM611 case).
   - `tlsBadUrl`: `node:https` server using a **committed** self-signed throwaway
     cert/key pair (CN `mcp-medic-test-invalid`, README note marking it a test
     artifact, path added to any secret-scanner allowlist). Node's built-in
     `crypto` cannot mint certificates — do not attempt runtime generation.
3. `spawn-fixture.ts`: `fixtureCmd(name, env?)` returns
   `{command: process.execPath, args: [absolute fixture path], env}` — used by tests
   to build `ServerEntry` objects.
4. Smoke tests: each stdio fixture starts under a raw SDK client from the test itself
   (not the probe engine) and exhibits its labeled behavior within 10 s; http fixture
   endpoints return their labeled statuses.

# Acceptance Criteria

- [ ] All seven stdio fixtures behave per spec in smoke tests on macOS/Linux/Windows
      CI (path resolution via `process.execPath`, no shebang reliance).
- [ ] `ok.mjs` handshake completes < 2 s in CI; `hang.mjs` provably never responds
      (smoke test with its own 1 s abort).
- [ ] http fixture starts/stops cleanly: `close()` awaited, every created
      `http.Server`/`https.Server` reports `listening === false`, and a follow-up
      request to a closed port rejects.
- [ ] No fixture requires external network access (loopback `127.0.0.1` only).

# Validation

`npm test -- --run tests/unit/fixtures`; CI matrix green (Windows especially).

# Dependencies

01 (SDK dependency present). Independent of 04–24.

# Non-goals

- No probe engine logic (26–28), no assertions about mcp-medic behavior — fixtures
  only.

# Design References

- DESIGN.md §17.1 layer 3; §10 MM6xx (behavior ↔ rule mapping)
