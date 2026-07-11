# Title

Rule engine, registry, and ignore/fail-on plumbing

# Summary

Implement `src/rules/engine.ts` and `src/rules/registry.ts`: the `Rule` contract,
deterministic evaluation over an `Inventory`, finding de-duplication and sorting,
`--ignore` suppression, `--fail-on` exit-level computation, and the memoized
filesystem probe service (`which`/`stat`) that static rules share.

# Context

DESIGN.md §9 defines the contract and execution order; §10 fixes the catalog the
registry must eventually hold. The engine must be finished before any rule issue
(20–24, 28–30) so those issues only write `Rule` objects + tests.

# Scope

- `src/rules/engine.ts`, `src/rules/registry.ts`, `src/rules/fs-service.ts`
- `src/core/errors.ts` — `UsageError extends Error` (`name = 'UsageError'`,
  `code: 'EUSAGE'`, prototype fixed for `instanceof`); the CLI (issue 34) maps it to
  exit 2
- `tests/unit/rules/engine.test.ts`

# Detailed Requirements

1. Code the `Rule` interface from DESIGN.md §9 verbatim, with `RuleInput` defined as:
   ```ts
   interface RuleInput {
     inventory: Inventory;
     entry?: ServerEntry;          // for appliesTo 'entry'
     file?: ConfigFileInfo;        // for appliesTo 'file'
     probes?: ProbeResult[];       // for appliesTo 'probe'
     online?: OnlineData;          // for appliesTo 'online' (typed in issue 29; use a placeholder interface here)
     fs: RuleFsService;            // memoized which/stat/exists
     env: NodeJS.ProcessEnv;
   }
   ```
2. `RuleFsService` (in `fs-service.ts`): `which(cmd): string | null` (PATH + PATHEXT
   semantics on win32; memoized), `exists(p)`, `stat(p)` (memoized), `isExecutable(p)`
   (POSIX x-bit; win32: extension-based). Injectable/mockable for tests.
3. `registerRule(rule)` / `getRules()` in `registry.ts`; duplicate rule id → throw at
   registration (startup-time defect).
4. `evaluate(inventory, opts: EvaluateOptions): { findings: Finding[]; engineNotes: string[] }`
   with
   ```ts
   interface EvaluateOptions {
     ignore?: string[];          // validated rule ids (raw CLI values; unknown → UsageError)
     probes?: ProbeResult[];     // presence enables probe rules
     online?: OnlineData;        // presence enables online rules
     fs?: RuleFsService;         // default: real service
     env?: NodeJS.ProcessEnv;    // default: process.env
   }
   ```
   `engineNotes` are merged into the report's visibility notes by the orchestrator
   and never affect `--fail-on`.
   - Order: file rules → entry rules → inventory rules → probe rules → online rules
     (probe/online sections run only when data present in `opts`).
   - Client filter, per `appliesTo`: `file`/`entry` rules only receive
     matching-client files/entries; `inventory`/`probe`/`online` rules with a
     `clients` array receive a filtered inventory view (non-matching servers hidden,
     probes filtered by their entry's client).
   - Adapters emit no findings; MM101/MM102 (issue 20) convert
     `ConfigFileInfo.parseError`/`.structuralIssues` data — the engine has no merge
     step for adapter findings.
   - De-dup key: `(rule, serverKey ?? '', location?.file ?? '', location?.pointer ?? '')`.
   - Sort (missing values normalized): severity desc (`severityRank`), then rule id
     asc, then `location.file ?? ''` asc, then `location.line ?? Infinity` asc
     (findings without a line sort after numbered ones), then `serverKey ?? ''` asc
     — total and stable.
   - Suppression: `opts.ignore` (unknown id → throw `UsageError`) marks matches
     `suppressed: true` — they remain in the array.
   - Rules must be pure w.r.t. `RuleInput`; the engine wraps each `evaluate` in
     try/catch — a throwing rule appends `rule <id> crashed: <message>` to
     `engineNotes` (no synthetic Finding, no effect on exit codes) and never kills
     the run.
5. `computeExitLevel(findings, failOn): 0 | 1` — counts non-suppressed findings with
   `severityRank >= rank(failOn)`; `failOn: 'never'` always 0.
6. `registry.ts` also exports `ruleTable(): Array<{id, category, defaultSeverity, summary, clients?}>`
   — single source for the `rules` CLI command and the README generator (issue 36).

# Acceptance Criteria

- [ ] Synthetic-inventory test with 3 stub rules proves ordering, de-dup, suppression,
      and crash-isolation behaviors.
- [ ] `computeExitLevel`: matrix test over {error,warn,info,never} ×
      {errors-only, warns-only, infos-only, none, suppressed-only} → documented values
      (suppressed findings never trip the threshold).
- [ ] `which('node')` resolves on all three CI OSes; `which('definitely-missing-xyz')`
      → null; memoization asserted (fs spy called once for repeated queries).
- [ ] Unknown `--ignore` id raises `UsageError`.

# Validation

`npm test -- --run tests/unit/rules/engine.test.ts` on the CI matrix (win32 PATHEXT
branch covered).

# Dependencies

04.

# Non-goals

- No actual rules (issues 20–24, 28–30), no report assembly (31), no CLI flags parsing
  (34 passes validated options in).

# Design References

- DESIGN.md §9 (engine), §2.2/§2.3 (`--ignore`, `--fail-on`), §10 (catalog shape)
