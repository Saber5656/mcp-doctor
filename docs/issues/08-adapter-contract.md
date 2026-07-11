# Title

ClientAdapter contract, registry, and shared contract-test harness

# Summary

Implement the `ClientAdapter` interface, the adapter registry, the discovery step of
the orchestrator (candidate files → loaded files → entries), and a reusable
contract-test harness + fixture conventions that every adapter issue (09–18) plugs
into.

# Context

DESIGN.md §7.1 defines the contract and the six adapter obligations, plus the
"adapters record facts; rules emit findings" principle. Eight real client adapters
(issues 09–17) and the declarative custom-adapter loader (issue 18) will be written
in parallel by different agents; the harness is what keeps them uniform and honest.

# Scope

- `src/discovery/adapter.ts` (interface, registry, `discoverAll()` driver)
- `tests/harness/adapter-contract.ts` (exported test factory)
- `tests/fixtures/homes/README.md` (fixture-layout convention doc)

# Detailed Requirements

1. Code the interfaces from DESIGN.md §7.1 verbatim (`ClientAdapter`, `CandidateFile`,
   `LoadedFile`, `AdapterParseResult`, `DiscoveryContext`).
2. `registerAdapter(a: ClientAdapter)` + `getAdapters(filter?: ClientId[])`; built-ins
   self-register from `src/discovery/adapters/index.ts` (created here with an empty
   list; adapter issues append).
3. `discoverAll(ctx, adapters): { configFiles: ConfigFileInfo[]; servers: ServerEntry[]; notes: string[] }`:
   - For each adapter: `candidateFiles(ctx)` → for each: `loadTextFile` (issue 06) →
     `ConfigFileInfo` mapping: `missing` → `exists: false`;
     `too-large`/`unreadable`/`io-error` → `exists: true` + `loadError` (MM107
     input), file not parsed; `ok` → `exists: true` + size/mode captured, then
     `parseFile` inside try/catch (a throwing adapter is a bug: catch, convert to
     note `adapter <id> crashed: <msg>`, continue). `parseError`/`structuralIssues`
     from `AdapterParseResult` copied onto the `ConfigFileInfo`; `notes` merged into
     the discovery-level notes. For `json`/`jsonc` files that loaded, `discoverAll`
     additionally runs `findDuplicateKeys` (issue 06) over the whole file and stores
     the result in `ConfigFileInfo.duplicateKeys` (MM108 input) — generic, so
     adapters don't repeat it.
   - Symlink policy (DESIGN §14.2 case 4), applied to candidates with
     `origin: 'builtin'` and `scope: 'user'` only (`origin: 'custom'` — set by the
     issue-18 loader — and project-scope files are exempt): compute
     `rel = path.relative(await realpath(home()), await realpath(candidate))`; skip
     with a note when `rel` starts with `..` or is absolute. Never rely on string
     prefix comparison (`/tmp/home2` must not pass a `/tmp/home` root check).
   - Assign `ServerEntry.key` via `makeServerKey` with a per-run `taken` set.
4. Contract-test harness `adapterContractTests(adapter, fixtureDir)`:
   - **Fixture runner contract**: each case directory contains `home/` (fake home
     tree), optional `project/` (used as `projectDir`), optional `case.json`
     (`{platform?: NodeJS.Platform, env?: Record<string,string>}`), and
     `expected.json`. The harness builds `DiscoveryContext` with
     `env = {MCP_MEDIC_HOME: <abs home/>, APPDATA: <abs home/AppData/Roaming>,
     XDG_CONFIG_HOME: <abs home/.config>, ...case.json env}`, `platform` from
     `case.json` (default: current), `projectDir` = abs `project/` or a
     guaranteed-empty temp dir.
   - **`expected.json` schema**: `{ entries: Array<{ name, scope, transport,
     command?, args?, envKeys?, url?, enabled, clientSpecific?: object }> ,
     parseError?: boolean, structuralIssueKinds?: string[] }`. `clientSpecific` is
     subset-matched (only listed keys compared) so quirk fixtures can assert
     `unresolvedVars`, `transportAssumed`, `declaredType`, `secretCandidates`
     length, etc. Env **values** are never listed — only key names.
   - Standard cases every adapter must provide: `missing-file/` (0 entries,
     `exists:false`), `valid-empty/` (file exists, no servers key → 0 entries, no
     issues), `valid-full/`, `malformed/` (`parseError` set, no throw),
     `secrets-planted/` (planted values present **in memory** — redaction happens at
     output, not parse — and, where the adapter defines `secretCandidates`, asserted
     via the projection).
   - Universal assertions on every case: adapter never throws; every entry has an
     absolute `source.file`; every `source.pointer` is RFC 6901-escaped and actually
     resolves against the parsed file content (harness re-parses and resolves —
     obligation 3 of §7.1); no `Finding` objects anywhere in `AdapterParseResult`.
5. Fixture convention doc (`tests/fixtures/homes/README.md`) records the runner
   contract and `expected.json` schema above, plus the guidance that adapter issues
   add quirk cases (unresolved vars, transport assumptions) beyond the standard five.

# Acceptance Criteria

- [ ] A dummy in-repo test adapter (fixture-only, registered in tests) passes the
      harness end-to-end — including a quirk case asserting a `clientSpecific`
      subset-match and a pointer containing `~`/`/` characters (escaping verified) —
      proving the harness itself works before real adapters land.
- [ ] `discoverAll` with a crashing stub adapter yields a note, not a crash.
- [ ] LoadError mapping: fixtures for `unreadable` (chmod 000, POSIX-only) and
      `too-large` produce `exists: true` + correct `loadError.kind`, no parse
      attempt.
- [ ] Symlink-escape matrix (POSIX; skipped on win32): user-scope candidate
      symlinked outside the fake home → skipped with note; sibling-prefix directory
      (`<home>2/…`) correctly treated as outside; symlink **within** the home → not
      skipped.
- [ ] Key collision inside one parse produces `#2` suffix and both entries survive.

# Validation

`npm test -- --run tests/unit/discovery tests/harness`; reviewer verifies all six
obligations of DESIGN.md §7.1 each have at least one harness assertion (env-expansion
and transport outcomes are assertable through the `expected.json` `clientSpecific` /
`transport` projections; adapter issues supply those fixtures).

# Dependencies

04, 05, 06.

# Non-goals

- No real client adapters (09–17), no custom-adapter loader (18), no rules.

# Design References

- DESIGN.md §7.1 (contract), §14.2 case 4 (symlinks), §17.1 layer 2 (contract tests)
