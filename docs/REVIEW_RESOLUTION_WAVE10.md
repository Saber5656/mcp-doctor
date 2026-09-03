# Wave 10 concrete review-resolution addendum

Repository: Saber5656/mcp-doctor
Pull request: #1

This addendum is the normative response to the 38 mapped human review findings on
this documentation-only pull request. Every finding has one implementation path
and one focused verification gate. The independent review manifest pins the
current head, base, and addendum blob; mutable identities are intentionally not
copied here. A head or base change invalidates this document and requires a fresh
review.

These contracts do not claim that implementation, tests, security review, or
release validation have already passed. A thread is eligible for resolution only
after its focused gate, repository full validation, and the applicable
security/privacy handoff have terminal success for the pinned identity. No PR
review bot is triggered or rerun.

## Mandatory completion gates

The implementation owner must attach focused-test and full-repository evidence.
Security/privacy review must cover config reads, symlink containment, subprocess
and network consent, secret redaction, and release permissions where applicable.
Missing, pending, skipped, cancelled, timed-out, stale, or unknown evidence blocks
resolution. The final merge gate separately re-fetches policy, checks, reviews,
head/base, and unresolved threads.

## Thread contracts

### 1. Guard --output against client config targets

- Thread: PRRT_kwDOTNj9Nc6QDHdX
- Location: docs/decisions/ADR-005:18
- Normative resolution: Before writing a report, the CLI resolves the output
  target and every discovered client-config target through existing ancestors and
  rejects a target whose canonical identity equals or aliases a client config.
  The rejection occurs before opening the output file.
- Focused gate: Use direct, relative, and symlinked output paths targeting a
  discovered config and assert the same refusal with unchanged config bytes.

### 2. Clarify output format does not disable probes

- Thread: PRRT_kwDOTNj9Nc6QDHda
- Location: docs/DESIGN.md:703
- Normative resolution: Format and destination affect rendering only. Probe
  eligibility and execution depend solely on --connect/--online, consent, client
  selection, and timeout options, so text, JSON, and Markdown run identical probes.
- Focused gate: Run each format with a counting fake probe under identical options
  and assert equal probe requests and only renderer differences.

### 3. Resolve published dependency strategy before implementation

- Thread: PRRT_kwDOTNj9Nc6QDHdc
- Location: docs/DESIGN.md:882
- Normative resolution: mcp-medic publishes runtime dependencies as external npm
  dependencies in package.json and does not bundle them into the application
  artifact. The lockfile pins them, npm audit covers the installed tree, and the
  package smoke test installs from the packed artifact.
- Focused gate: Inspect package.json, lockfile, and npm pack contents, install the
  packed package in a clean fixture, and assert runtime imports resolve without
  bundling a second copy.

### 4. Add actions/checkout before CodeQL init

- Thread: PRRT_kwDOTNj9Nc6QDHdd
- Location: docs/issues/03-community-security-files.md:53
- Normative resolution: The CodeQL job performs SHA-pinned actions/checkout before
  github/codeql-action/init and before any source-dependent analysis step.
- Focused gate: Parse the workflow job order and assert checkout precedes CodeQL
  init with an immutable commit SHA.

### 5. Fix short-secret masking contradiction

- Thread: PRRT_kwDOTNj9Nc6QDHde
- Location: docs/issues/07-redaction-module.md:47
- Normative resolution: A secret shorter than eight characters produces no masked
  sample; otherwise the mask length is min(5, floor(length / 4)). The same formula
  is used by report and log renderers.
- Focused gate: Test lengths 0, 7, 8, 19, and 20 and assert the exact mask length
  in both output paths.

### 6. Do not call realpath() on missing candidates

- Thread: PRRT_kwDOTNj9Nc6QDHdf
- Location: docs/issues/08-adapter-contract.md:49
- Normative resolution: Path resolution calls realpath only for an existing
  candidate. For a missing candidate it resolves the nearest existing ancestor,
  appends the normalized missing suffix, and returns exists: false without
  opening or realpath-ing the missing leaf.
- Focused gate: Resolve existing, missing, and missing-under-symlink fixtures and
  assert the returned exists flag, canonical ancestor, and zero missing-leaf read.

### 7. Define settings-file precedence explicitly

- Thread: PRRT_kwDOTNj9Nc6QDHdg
- Location: docs/issues/11-adapter-claude-code-project-managed.md:52
- Normative resolution: Claude Code settings precedence is managed-settings.json,
  project settings.local.json, project settings.json, then user settings.json.
  The first matching file wins; within that file disabledMcpjsonServers wins over
  enabledMcpjsonServers, which wins over enableAllProjectMcpServers. The deciding
  path is recorded in enabledSource.
- Focused gate: Supply conflicting values in all four files and assert the managed
  decision, then remove it and assert the exact next winner at each step.

### 8. Fail closed when a user-scoped path remains relative

- Thread: PRRT_kwDOTNj9Nc6QDHdi
- Location: docs/issues/18-adapter-custom.md:52
- Normative resolution: A user-scoped custom path that remains relative after
  adapter normalization yields MM105 with an invalid-user-path detail and is not
  read, probed, or included as a usable candidate.
- Focused gate: Provide a relative user path and assert the finding plus zero
  filesystem reads for the candidate.

### 9. Filter inventory for every client-scoped rule

- Thread: PRRT_kwDOTNj9Nc6QDHdj
- Location: docs/issues/19-rule-engine.md:63
- Normative resolution: Before evaluating any client-scoped rule, the rule engine
  passes only inventory entries whose client id equals the rule's client scope.
  Cross-client rules explicitly opt into the complete inventory.
- Focused gate: Place matching entries in two clients and assert a client-scoped
  rule reports only its selected client while a cross-client rule reports both.

### 10. Preserve location of unresolved variables

- Thread: PRRT_kwDOTNj9Nc6QDHdk
- Location: docs/issues/21-rules-broken-static.md:47
- Normative resolution: Every unresolved variable finding carries unresolvedVars
  entries with name and field, where field identifies command, args[index], or
  env[key]. The original entry location remains attached to the finding.
- Focused gate: Use unresolved values in command, argument, and environment fields
  and assert the exact name/field pairs and source location.

### 11. Make raw-command identities unambiguous

- Thread: PRRT_kwDOTNj9Nc6QDHdm
- Location: docs/issues/22-rules-duplicates.md:31
- Normative resolution: A raw-command identity is JSON.stringify of the ordered
  two-element array [normalizedCommand, args]. Command and argument boundaries
  are therefore preserved and no delimiter concatenation is used.
- Focused gate: Compare command/argument pairs that would collide under
  concatenation and assert distinct identities plus stable equality for repeats.

### 12. Align MM502 with shared transport normalization rules

- Thread: PRRT_kwDOTNj9Nc6QDHdn
- Location: docs/issues/24-rules-hygiene.md:37
- Normative resolution: MM502 evaluates every supported client transport after the
  shared normalization stage, including clients whose normalized endpoint can
  expose SSE. It applies the same normalized endpoint/header predicate to each
  client and records the client id in the finding.
- Focused gate: Build one normalized SSE-capable entry for every supported client
  fixture and assert MM502 coverage and identical predicate behavior.

### 13. Include EPERM in spawn-failure classification

- Thread: PRRT_kwDOTNj9Nc6QDHdo
- Location: docs/issues/26-probe-stdio.md:78
- Normative resolution: Spawn errors ENOENT, EACCES, and EPERM all map to the
  structured MM601 spawn-failed result with errno preserved and no retry that
  executes an unapproved command.
- Focused gate: Inject each errno into the spawn adapter and assert MM601, the
  matching errno, and one bounded spawn attempt.

### 14. Resolve consent-error versus never-rejecting run contract

- Thread: PRRT_kwDOTNj9Nc6QDHdq
- Location: docs/issues/28-probe-runner.md:32
- Normative resolution: createProbeRunner validates consent synchronously and
  returns a typed configuration error before probing when consent is absent.
  Once created, runner.run always resolves a ProbeResult for every candidate;
  target failures are result statuses and never rejected promises.
- Focused gate: Call creation without consent and assert the synchronous error;
  then run spawn, timeout, and handshake failures and assert resolved results for
  all candidates.

### 15. Do not convert every PathResolutionError into global exit 2

- Thread: PRRT_kwDOTNj9Nc6QDHdt
- Location: docs/issues/34-cli-wiring.md:65
- Normative resolution: Invalid global CLI/configuration paths return exit 2.
  A path error for one discovered client becomes a client-scoped finding and the
  scan continues for all other clients.
- Focused gate: Run one invalid client path beside one valid client and assert
  completed report, per-client finding, valid-client results, and exit mapping.

### 16. Add secrets to fixture manifest and make assertions mode-aware

- Thread: PRRT_kwDOTNj9Nc6QDHdu
- Location: docs/issues/35-e2e-suite.md:28
- Normative resolution: Each fixture manifest declares secrets[] for redaction
  vectors. Static assertions identify expectedFile; connect assertions identify
  serverKey; online assertions identify endpoint and predicate. The harness selects
  the required identity fields from the fixture mode and never compares a static
  file location to a network result.
- Focused gate: Run static, connect, and online fixtures and assert secrets
  coverage plus the mode-specific identity predicate for each finding.

### 17. Use one authoritative rule catalog for coverage

- Thread: PRRT_kwDOTNj9Nc6QDHdv
- Location: docs/issues/35-e2e-suite.md:41
- Normative resolution: A build step generates one rule catalog from the DESIGN
  rule definitions; runtime registry metadata and E2E coverage both consume that
  catalog. CI fails when generated catalog and DESIGN-derived source differ.
- Focused gate: Regenerate the catalog, compare runtime and fixture rule ids, and
  mutate one DESIGN rule to assert the drift check fails.

### 18. Scope version normalization to tool-version field

- Thread: PRRT_kwDOTNj9Nc6QDHdw
- Location: docs/issues/35-e2e-suite.md:65
- Normative resolution: Version normalization transforms only the structured
  toolVersion field. Command lines, URLs, server names, finding details, and other
  strings are compared verbatim after normal redaction/sanitization.
- Focused gate: Place version-like text in every non-version field and assert only
  toolVersion changes during golden normalization.

### 19. Gate publishing on event type and boolean input

- Thread: PRRT_kwDOTNj9Nc6QDHdy
- Location: docs/issues/37-release-automation.md:33
- Normative resolution: Publishing runs only when github.event_name is push, the
  ref is a version tag, and inputs.dry_run is the boolean false. workflow_dispatch
  always executes the dry-run path.
- Focused gate: Exercise tag push, manual false, manual true, and branch push
  events and assert publish occurs only for the tag-push case.

### 20. Treat managed MCP as exclusive

- Thread: PRRT_kwDOTNj9Nc6QDKf6
- Location: docs/issues/11-adapter-claude-code-project-managed.md:57
- Normative resolution: When managed-mcp.json exists, its server inventory is the
  authoritative managed scope and non-managed duplicate entries for the same
  managed identity are marked inventory-only and are not probed.
- Focused gate: Provide a managed entry and a duplicate project/user entry and
  assert only the managed identity is probe-eligible.

### 21. Flag JSONC-only syntax for all strict JSON clients

- Thread: PRRT_kwDOTNj9Nc6QDKf8
- Location: docs/issues/06-config-readers.md:18
- Normative resolution: Strict JSON parsing rejects comments and trailing commas
  for Claude, Codex, Cursor, Windsurf, Gemini, and every other plain-JSON client.
  Only the explicitly documented VS Code and Zed readers use JSONC parsing.
- Focused gate: Run comment and trailing-comma fixtures through every client
  reader and assert rejection for strict readers and acceptance only for JSONC
  readers.

### 22. Sanitize JSON reports before stringifying

- Thread: PRRT_kwDOTNj9Nc6QDKf9
- Location: docs/issues/31-report-json.md:49
- Normative resolution: Before JSON serialization, a recursive sanitizer visits
  every string in inventory, probes, findings, locations, details, and metadata,
  removes control characters, and applies the redaction policy. No server-only
  field is exempt from the traversal.
- Focused gate: Put control characters and secret sentinels in each report branch
  and assert sanitized JSON contains neither raw control bytes nor sentinels.

### 23. Resolve commands with the entry PATH

- Thread: PRRT_kwDOTNj9Nc6QDKf_
- Location: docs/issues/21-rules-broken-static.md:42
- Normative resolution: Command resolution uses an effective PATH formed from the
  entry's PATH value when present, followed by the documented process PATH
  fallback. The effective PATH is used only for that entry's resolution.
- Focused gate: Supply two fake binaries visible only through entry.env.PATH and
  assert the rule resolves those binaries without changing another entry.

### 24. Quote argv tokens in consent plan

- Thread: PRRT_kwDOTNj9Nc6QDKgC
- Location: docs/issues/34-cli-wiring.md:48
- Normative resolution: Consent output renders argv as JSON.stringify of the exact
  token array, with each boundary and special character preserved. It never joins
  tokens into an unquoted shell command.
- Focused gate: Render argv containing spaces, quotes, and shell metacharacters and
  assert the consent text contains the exact JSON array and no executable shell
  line.

### 25. Do not dedupe probes with duplicate-report identity

- Thread: PRRT_kwDOTNj9Nc6QDKgF
- Location: docs/issues/28-probe-runner.md:48
- Normative resolution: Probe deduplication identity includes transport, command,
  ordered args, cwd, and all behavior-affecting environment values. The report
  serverIdentity is used only for reporting and never suppresses a behaviorally
  distinct probe.
- Focused gate: Use same server identity with different args/cwd/env and assert
  separate probes; use identical behavior with different report labels and assert
  one probe.

### 26. Support Gemini default env expansions

- Thread: PRRT_kwDOTNj9Nc6QDKgI
- Location: docs/issues/17-adapter-gemini.md:51
- Normative resolution: Gemini expansion of ${VAR:-DEFAULT} yields DEFAULT when
  VAR is unset or empty; ${VAR} with no default yields an unresolved-variable
  finding when VAR is unset.
- Focused gate: Run set, unset, empty, and defaulted variable fixtures and assert
  expanded args and unresolved findings exactly.

### 27. Do not mark unset Windsurf env interpolation as unresolved

- Thread: PRRT_kwDOTNj9Nc6QDKgJ
- Location: docs/issues/14-adapter-windsurf.md:46
- Normative resolution: Windsurf ${env:NAME} interpolation yields an empty string
  when NAME is unset and does not produce an unresolved-variable finding.
- Focused gate: Parse set and unset interpolation fixtures and assert value
  substitution plus no unresolved finding for the unset case.

### 28. Include envFile values in stdio probes

- Thread: PRRT_kwDOTNj9Nc6QDKgM
- Location: docs/issues/26-probe-stdio.md:37
- Normative resolution: Under --connect consent, the adapter loads envFile using
  safe parsing, merges values with explicit entry env taking precedence, and
  supplies the result to the child. Secret values are never copied to reports or
  consent text.
- Focused gate: Probe an envFile-only variable and an explicit override, assert
  child environment precedence, and scan all output for the secret sentinel.

### 29. Use Codex bearer token env var for HTTP probes

- Thread: PRRT_kwDOTNj9Nc6QDKgN
- Location: docs/issues/12-adapter-codex.md:43
- Normative resolution: With --connect consent, the Codex HTTP probe reads the
  configured bearer token from the approved environment variable and sends it as
  an Authorization Bearer header. The token is redacted before any log or report.
- Focused gate: Run a fake endpoint with a token-bearing fixture and assert the
  header is received while the token is absent from consent, report, and logs.

### 30. Probe explicit SSE endpoints with SSE transport

- Thread: PRRT_kwDOTNj9Nc6QDKgP
- Location: docs/issues/27-probe-http.md:36
- Normative resolution: An endpoint explicitly normalized as SSE is probed first
  with SSEClientTransport. The result records sse transport and does not silently
  reinterpret an explicit SSE endpoint as streamable HTTP.
- Focused gate: Use an SSE fixture that rejects streamable HTTP and assert one
  SSEClientTransport attempt with the expected result.

### 31. Correct network-I/O security promise

- Thread: PRRT_kwDOTNj9Nc6QDKgQ
- Location: docs/issues/03-community-security-files.md:40
- Normative resolution: Static scan performs no network I/O. --connect permits only
  configured server endpoints after consent, while --online permits only the npm
  registry package-name lookups; the two permissions are independent.
- Focused gate: Run static, connect-only, online-only, and both modes with network
  spies and assert the exact allowed request classes.

### 32. Check symlink escapes before loading files

- Thread: PRRT_kwDOTNj9Nc6QDKgS
- Location: docs/issues/08-adapter-contract.md:36
- Normative resolution: The adapter resolves and checks canonical containment
  beneath the configured client/project root before opening any file. A symlink
  escape returns a path finding and performs no file read.
- Focused gate: Place a candidate symlink to an outside file and assert containment
  refusal, unchanged outside bytes, and zero read of the target.

### 33. Exercise publish in release dry-runs

- Thread: PRRT_kwDOTNj9Nc6QDKgU
- Location: docs/issues/37-release-automation.md:31
- Normative resolution: workflow_dispatch always runs npm publish --dry-run and
  validates the packed artifact; a real publish is reachable only from the
  approved version-tag push condition.
- Focused gate: Dispatch the workflow with dry-run input and assert the exact
  --dry-run command; inspect the tag path and assert it is the only real publish
  path.

### 34. Include nested Codex project config layers

- Thread: PRRT_kwDOTNj9Nc6QDKgW
- Location: docs/issues/12-adapter-codex.md:30
- Normative resolution: Codex discovery reads project config layers from project
  root toward cwd and applies the closest layer last, so the nearest file wins
  for a duplicate key while unrelated keys merge deterministically.
- Focused gate: Create root, nested, and cwd config fixtures with conflicting
  values and assert nearest-wins plus stable merged output.

### 35. Show actual HTTP probe effects in consent

- Thread: PRRT_kwDOTNj9Nc6QDKgX
- Location: docs/issues/34-cli-wiring.md:48
- Normative resolution: The consent plan displays HTTP method, a redacted URL,
  header key names, and the selected SSE/streamable-HTTP fallback sequence. It
  never displays header values or executable dynamic-header commands.
- Focused gate: Render stdio, HTTP, and SSE consent fixtures and assert method,
  redacted URL, key names, and fallback text with all secret values absent.

### 36. Honor Codex stdio cwd and env whitelists

- Thread: PRRT_kwDOTNj9Nc6QDKgY
- Location: docs/issues/12-adapter-codex.md:38
- Normative resolution: Codex stdio probe invocations carry the adapter-resolved
  cwd and the approved env_vars allowlist into the child request. No ambient
  environment value outside that allowlist is inherited.
- Focused gate: Use a fixture with a custom cwd, one allowed variable, and one
  ambient secret and assert the child receives only the cwd and allowed variable.

### 37. Apply client tool filters before MM610

- Thread: PRRT_kwDOTNj9Nc6QDKga
- Location: docs/issues/28-probe-runner.md:59
- Normative resolution: include/exclude client and tool filters are applied before
  MM610 capability evaluation. A filtered-out tool cannot produce an MM610 finding
  or affect tool-count facts.
- Focused gate: Run a fixture where only a filtered tool lacks capability and
  assert no MM610; remove the filter and assert MM610 appears.

### 38. Do not cap Codex probes below client timeout

- Thread: PRRT_kwDOTNj9Nc6QDKgb
- Location: docs/issues/26-probe-stdio.md:65
- Normative resolution: The effective Codex probe timeout is the greater of the
  client-configured timeout and the user diagnostic timeout when one is supplied.
  The effective value is recorded in probe facts and never reduced below the
  client contract.
- Focused gate: Run client timeouts 5s and 15s with diagnostic values 1s and 20s
  and assert effective values 5s, 15s, 20s, and the recorded timeout facts.

## Resolution boundary

The implementation owner must update affected design and issue text consistently,
attach evidence to every mapped thread, and resolve only after focused,
full-validation, and specialist gates pass for the pinned identity.
