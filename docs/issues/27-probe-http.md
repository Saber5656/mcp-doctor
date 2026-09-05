# Title

HTTP/SSE probe engine

# Summary

Implement `src/probe/http.ts` + the HTTP half of `classify.ts`: probe remote MCP
endpoints via streamable HTTP with one legacy-SSE fallback, classify
DNS/TCP/TLS/HTTP-status/protocol outcomes into MM606/MM607/MM611, honoring static
headers and never executing dynamic header helpers.

# Context

DESIGN.md §11.3. Remote entries (claude-code http/sse, codex url, cursor/windsurf
remote, gemini url/httpUrl, vscode http) get connectivity + handshake verification.
Auth-protected servers are healthy-but-unverifiable (MM607 info), not errors.

# Scope

- `src/probe/http.ts` — `probeHttp(entry, opts): Promise<ProbeResult>`
  (classification into `status` + `facts.httpClass`/`authRejected`/
  `fallbackTransport` per DESIGN §4)
- `src/rules/probe/http-rules.ts` — `Rule` objects (appliesTo `probe`) MM606,
  MM607 (protocol-invalid 2xx bodies surface as status `handshake-error` and are
  converted by issue 26's transport-agnostic MM603 rule)
- `tests/integration/probe/http.test.ts` (against issue 25 http fixture) +
  `tests/unit/rules/probe/http-rules.test.ts`

# Detailed Requirements

1. Transport attempt order:
   1. SDK `StreamableHTTPClientTransport` + `Client.initialize` against `entry.url`
      with `entry.headers` (values used as-is; they are the user's own credentials
      contacting the user's own endpoint).
   2. On HTTP 4xx/405/406 from step 1 **and** entry transport is `sse` or `unknown`
      (never downgrade an explicit `http` entry): one attempt with legacy
      `SSEClientTransport`. Record which transport succeeded in
      `ProbeResult.detail`.
2. Controls: `AbortSignal.timeout(opts.timeoutMs)` per attempt (plus `opts.signal`
   from the runner, same abort semantics as issue 26); redirects via a **custom
   fetch wrapper** injected into the SDK transport options
   (`redirect: 'manual'`): follow up to 3 same-origin redirects manually; a
   cross-origin `Location` stops the probe and classifies
   `facts.httpClass: 'redirect'`. TLS verification always on — there is no insecure
   flag anywhere in the codebase.
3. Post-handshake: `tools/list` (same semantics as stdio; toolNames returned).
4. Classification into `status` + `facts.httpClass`; MM606/MM607 rules convert:
   | Observation | status + facts | Rule → Finding |
   |---|---|---|
   | DNS resolution failure | `http-error`, `httpClass: 'dns'` | MM606 |
   | TCP refused / reset / timeout | `http-error`, `httpClass: 'connect'` / `'timeout'` | MM606 |
   | TLS handshake/cert error | `http-error`, `httpClass: 'tls'` | MM606 (+ remediation: fix the cert; mcp-medic offers no bypass) |
   | HTTP 401/403 (either transport) | `auth-required`; `facts.authRejected: true` when a static `Authorization` header was configured and still rejected | MM607 (info): "requires interactive auth (e.g. OAuth); not verifiable non-interactively"; `authRejected` adds "the configured credential was rejected" (severity stays info; the credential value never appears) |
   | HTTP 404 / 410 | `http-error`, `httpClass: 'not-found'` | MM606 (likely wrong path) |
   | HTTP 5xx | `http-error`, `httpClass: 'server-error'` | MM606 |
   | 2xx but protocol-invalid body / initialize failure | `handshake-error` | MM603 (issue 26's transport-agnostic rule) |
   | cross-origin redirect | `http-error`, `httpClass: 'redirect'` | MM606 (target origin shown, path redact-checked) |
   | ok + old protocolVersion | `ok`, `protocolVersion` | MM611 (issue 26's rule) |
5. `headersHelper`/dynamic header mechanisms: never executed (ADR-003); when present,
   append to MM607/MM606 detail: "dynamic headers (headersHelper) were not executed".
6. URL handling: probe exactly `entry.url` (no path guessing, no `/.well-known`
   walking in v1). Error `detail` strings pass `maskUrl` + sanitize.
7. `probeHttp` never throws.

# Acceptance Criteria

- [ ] Fixture matrix: ok→ok (+tools); 401→`auth-required`+MM607(info);
      404→MM606(not-found); 500→MM606(server-error); tlsBad→MM606(tls);
      `slowStart` with `HTTP_SLOW_MS` ≫ 1 s probe timeout→MM606(timeout — fully
      local, deterministic); NXDOMAIN (`*.invalid` reserved TLD)→MM606(dns);
      `oldProtocol`→ok+MM611.
- [ ] SSE fallback: `sseFallback` fixture (405 streamable, working SSE) → probed ok
      with `facts.fallbackTransport: 'sse'`; explicit `http` entry never falls back
      (asserted).
- [ ] Configured fake bearer header reaches the fixture — asserted via the
      `auth-header-seen` tool name appearing in `tools/list` (issue 25 mechanism);
      the header value is absent from all findings/details.
- [ ] No insecure-TLS code path exists (`rejectUnauthorized` / `NODE_TLS_REJECT_...`
      absent from src — static grep test).

# Validation

`npm test -- --run tests/integration/probe/http.test.ts` on CI matrix.

# Dependencies

04, 07 (masking/sanitization), 19 (rule registration), 25 (http fixtures).

# Non-goals

- No OAuth flows, no ws probing (v1 skip per §11.1), no endpoint discovery, no
  header-helper execution.

# Design References

- DESIGN.md §11.3, §2.4 item 5; §10 MM606/MM607/MM611
