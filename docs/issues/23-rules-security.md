# Title

Security rules MM401–MM410

# Summary

Implement the ten static security rules: plaintext secrets (env/headers/auth buckets),
loose file permissions, non-TLS URLs, unpinned package execution, dynamic-exec hooks,
shell wrappers, secrets in argv, credentials in URLs, Gemini trust bypass, and envFile
issues.

# Context

DESIGN.md §10 (Security table) + §12 (detection/masking) + §14. The redaction module
(issue 07) provides `detectSecret`/`mask`/`maskUrl`; these rules provide the policy.
Adapters feed `clientSpecific.secretCandidates` (see issue 13's documented pattern)
for secret material living at non-standard paths (e.g. Cursor `auth.CLIENT_SECRET`).

# Scope

- `src/rules/security/mm401.ts` … `mm410.ts` (+ barrel)
- `tests/unit/rules/security/*.test.ts` (fixtures with planted fake secrets)

# Detailed Requirements

Common: **evidence strings in findings must already be masked** — call
`mask`/`maskUrl` inside the rule; never place a raw candidate into `Finding.detail`.
Tests assert the raw value is absent from every finding field.

1. **MM401** (entry, error): `detectSecret(value, key)` hits over: `env` values,
   `headers` values, `clientSpecific.secretCandidates[].value`. One finding per hit,
   pointer to the exact field. Skip rule: values consisting entirely of a VS Code
   input reference (`/^\$\{input:[^}]+\}$/`) are never scanned (shared exemption
   contract with issue 15 — regression fixture required).
   Remediation per client: env-var indirection (`${VAR}` claude-code, `${env:VAR}`
   windsurf, `bearer_token_env_var` codex, `inputs` vscode) — a small
   per-client remediation map lives in this module.
2. **MM402** (file, error/info): POSIX only. Preconditions: file contains ≥1 MM401 hit
   → severity error when `posixMode & 0o077 ≠ 0` ("contains secrets and is readable
   by group/others: chmod 600"). No secrets but mode loose → info variant. win32:
   inert (document ACL check as v2).
3. **MM403** (entry, error): transport ∈ {`http`, `sse`, `ws`} and URL scheme is
   `http:` or `ws:` and hostname ∉ {localhost, 127.0.0.0/8, ::1, *.localhost}.
   Detail: token material and tool traffic would transit cleartext.
4. **MM404** (entry, warn): `extractPackageSpec(command, args)` (issue 21,
   `src/core/pkgspec.ts`) returns a `PkgSpec` with `pinned === false` (covers bare,
   `@latest`, ranges). Detail explains supply-chain drift of `npx -y`; remediation
   shows the pinned form with the currently-latest placeholder text (no network:
   `pkg@<version>` literal guidance).
5. **MM405** (entry, info): `clientSpecific.headersHelper` present (claude-code).
   Neutral wording: legitimate feature, executes a shell command at client startup.
   `headersHelper` is a shell command **string**, not necessarily a path — only when
   the whole string is a single absolute path with no whitespace or shell
   metacharacters (`| & ; < > $ \` ( )`), stat it and, if group/world-writable,
   enrich the detail ("helper script is writable by others"); otherwise emit the
   info finding without any filesystem access.
6. **MM406** (entry, warn): `command` basename ∈ {sh, bash, zsh, dash, ksh} with
   `-c` in args; `cmd`/`cmd.exe` with `/c`; `powershell`/`pwsh` with `-Command`.
   Remediation: replace with direct argv invocation of the real binary.
7. **MM407** (entry, warn): `detectSecret` hit inside any `args[i]` (argv is visible
   to `ps`/process explorers). Distinct from MM401 buckets; masked evidence.
8. **MM408** (entry, error): parsed `URL.password !== ''` (username-only userinfo
   does not fire), or a query parameter whose name matches
   `/key|token|secret|password|auth/i` and whose value trips `detectSecret`
   (keyHint = param name). Evidence via `maskUrl`.
9. **MM409** (entry, warn, clients: ['gemini-cli']): `clientSpecific.trust === true`.
   Detail: bypasses per-call confirmations for this server's tools.
10. **MM410** (entry, warn): `clientSpecific.envFile` present. Path resolution:
    absolute → as-is; `~` prefix → `expandTilde` (issue 05); relative → resolved
    against the directory of `entry.source.file`. Missing file → skip (MM205's
    turf). Existing file with POSIX mode `& 0o077 ≠ 0` → warn ("env file readable
    by group/others"). Tests cover absolute, `~`, and relative cases.

# Acceptance Criteria

- [ ] Planted-secret fixture matrix: every bucket (env, headers, secretCandidates,
      args, url-query, url-userinfo) both fires the right rule and yields fully
      masked evidence (raw substring absent from serialized findings — automated
      assertion, not eyeball).
- [ ] MM402 severity matrix: {secrets?, mode 600/644} × POSIX → {silent, info, error}
      as specified; win32 run emits nothing.
- [ ] MM403 negative cases: `https://`, `http://localhost:3000`, `http://127.0.0.1`.
- [ ] MM406 negative: plain `bash` script without `-c`; `sh` as a server binary name
      inside a path (`/opt/shell-server/sh-mcp` must not fire — basename+args check).
- [ ] Per-client remediation map covered by a test per client id.

# Validation

`npm test -- --run "tests/unit/rules/security/*.test.ts"`; reviewer confirms every
§10 MM4xx row implemented and every finding's evidence masked (grep planted values
over test output).

# Dependencies

19, 07, 21 (`extractPackageSpec` for MM404), 05 (`expandTilde` for MM410).

# Non-goals

- No dynamic verification (probe), no fixes/chmod (read-only, ADR-005), no network
  reputation checks (29/30), no Windows ACL analysis (v2).

# Design References

- DESIGN.md §10 Security (MM4xx), §12, §14.2 cases 1–3; ADR-003/ADR-005
