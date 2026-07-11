# Title

npm registry client + rules MM701/MM702/MM704

# Summary

Implement `src/online/npm.ts` — the opt-in registry metadata client (the only network
surface besides probes) — and the three registry-fact rules: package not found,
deprecated, and pinned-version-outdated.

# Context

DESIGN.md §11.5 and ADR-004: exactly one endpoint class
(`https://registry.npmjs.org/<name>`), package names only, bounded and cached.
Consumes `PkgSpec` values (issue 21) from stdio entries whose runner is npx/bunx
(`ecosystem === 'npm'`; pypi deferred to v2).

# Scope

- `src/online/types.ts` — exports `PkgMeta`, `NpmMetaStatus = PkgMeta | 'not-found' | 'unavailable'`,
  and `OnlineData = { npm: Map<string, NpmMetaStatus>; registryUnavailable: boolean }`
  (this replaces issue 19's placeholder; `RuleInput.online` imports from here)
- `src/online/npm.ts` — `fetchPackageMeta(names, opts): Promise<OnlineData>`
- `src/rules/online/mm701.ts`, `mm702.ts`, `mm704.ts` (+ barrel)
- `tests/unit/online/npm.test.ts` (mock server via `node:http` on 127.0.0.1)

# Detailed Requirements

1. Request: GET `https://registry.npmjs.org/<url-encoded name>` with
   `Accept: application/vnd.npm.install-v1+json` (abbreviated doc), UA
   `mcp-medic/<version>`, timeout 5 s, 2 retries with 300 ms backoff on network/5xx
   errors (404 is a result, not an error). The production base URL is hard-coded;
   `opts.registryBaseUrl` exists solely for tests targeting loopback mock servers
   (no CLI flag, no env var, no credentials — ADR-004's "exactly one endpoint
   class" stands).
2. Response handling: parse JSON; validate with a zod schema retaining only
   `{ name, 'dist-tags': {latest?}, versions: Record<string, {deprecated?: string}> , deprecated?: string }`
   (top-level `deprecated` appears on some fully-deprecated packages); everything else
   discarded. Any schema mismatch → `'unavailable'` for that package (never a crash).
   All retained strings pass `sanitizeForTerminal` (issue 07) and are hard-capped at
   300 chars — registry text reaches terminals.
3. Concurrency ≤ 4; per-run in-memory cache (Map) — no disk cache.
4. Total-failure semantics (DNS down / all requests failed): return all
   `'unavailable'` with `registryUnavailable: true` — that boolean is this issue's
   entire responsibility; the orchestrator (issue 34) turns it into the single
   visibility note "registry unreachable; online rules skipped". MM7xx rules treat
   `'unavailable'` as no-data (silent).
5. `OnlineData = { npm: Map<string, PkgMeta | 'not-found' | 'unavailable'> }` becomes
   `RuleInput.online` (placeholder from issue 19 replaced here).
6. Rules (appliesTo `online`, iterate entries with npm PkgSpec):
   - **MM701** (error): meta === `'not-found'`. Detail: the exact spec string and
     client/file; remediation: check spelling (and see MM703 for similar names).
   - **MM702** (warn): package-level `deprecated` present, or the **pinned** version's
     entry has `deprecated`. Deprecation message included after sanitize+mask pipeline
     (registry-controlled text ≤ 300 chars).
   - **MM704** (info): `pinned === true` and semver-major(pin) < semver-major(latest).
     Implement minimal semver-major extraction locally (`/^(\d+)\./`) — no semver dep.
7. Never called unless `--online`; module performs no I/O at import time.

# Acceptance Criteria

- [ ] Mock-registry tests: found/deprecated(pkg-level)/deprecated(version-level)/
      not-found/500-then-ok(retry)/timeout/malformed-json → exact statuses.
- [ ] MM701/702/704 firing + non-firing tests over synthetic OnlineData; MM704
      boundary: 1.9.0 vs 1.10.0 latest → silent; 1.x vs 2.x → fires.
- [ ] Cache: same package in 3 entries → exactly 1 HTTP request (mock counts).
- [ ] Registry-total-failure path: zero findings, `registryUnavailable: true`
      returned (the orchestrator note itself is asserted in issues 34/35), exit
      code unaffected.
- [ ] No request contains anything but the package name in the path (mock asserts
      full URL + headers).

# Validation

`npm test -- --run tests/unit/online`; reviewer confirms endpoint/payload constraints
against ADR-004.

# Dependencies

19, 21, 07 (`sanitizeForTerminal` for registry strings).

# Non-goals

- No PyPI/Docker (v2), no download counts/popularity, no disk caching, no proxy
  support of any kind in v1 (Node's default fetch does not reliably honor
  `HTTP(S)_PROXY` env vars across Node 20/22/24 — make no claims about proxies;
  corporate-proxy support is a v2 candidate).

# Design References

- DESIGN.md §11.5; §10 MM701/702/704; ADR-004
