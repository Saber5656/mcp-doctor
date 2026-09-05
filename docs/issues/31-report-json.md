# Title

Report assembly, JSON renderer, and exported JSON schema

# Summary

Implement `src/report/report.ts` (assemble `Report` from pipeline outputs, apply
output-redaction, compute summary + exit level), `src/report/schema.ts` (zod schema
mirroring the model), the JSON renderer, and the build step that exports
`schemas/report.schema.json`.

# Context

DESIGN.md §4 (`Report`), §13 (json renderer, schema versioning). The JSON report is
the machine contract for CI users; its schema is committed and versioned by
`schemaVersion: 1`.

# Scope

- `src/report/report.ts`, `src/report/schema.ts`, `src/report/render/json.ts`
- Build wiring: `npm run build:schema` (zod → JSON Schema via `z.toJSONSchema`,
  zod 4 built-in) executed in `build`; `schemas/report.schema.json` committed
- `tests/unit/report/{report,schema,json}.test.ts`

# Detailed Requirements

1. `assembleReport(input): Report` where input =
   `{inventory, findings, probes, extraNotes, mode, failOn, startedAt, toolVersion, clock?: () => Date}`:
   - Deep-copies inventory, applies `redactEntryForOutput` to **every** `ServerEntry`
     (issue 07), and **deletes the `raw` field from every entry** — `raw` never
     enters a `Report` in any form (DESIGN §4 says it is never printed; the report
     schema for server entries therefore has no `raw` property). The assembled
     Report is safe-by-construction; renderers add sanitization only.
   - `extraNotes` (engine notes + orchestrator notes such as
     "registry unreachable…", merged by the caller — issue 34) are appended to
     `inventory.visibilityNotes`.
   - `summary` counts exclude suppressed findings from error/warn/info tallies and
     reports them in `suppressed`.
   - `run.timestamp` = `startedAt` (ISO-8601 UTC);
     `durationMs = clock().getTime() - startedAt.getTime()` with `clock` defaulting
     to `() => new Date()` — tests inject a fixed clock.
   - Deterministic ordering (findings pre-sorted by engine; servers sorted by key;
     configFiles by path) — byte-identical JSON for identical inputs given a fixed
     clock.
2. `schema.ts`: zod schemas for every §4 interface, `strict()` objects (unknown keys
   rejected) — the schema is both a validator (used in tests + by the E2E suite) and
   the source of the exported JSON Schema. `schemaVersion` literal `1`.
3. JSON renderer: `renderJson(report): string` — `JSON.stringify(report, null, 2)`;
   guarantees trailing newline; no ANSI ever.
4. Schema export: build script generates draft-2020-12 JSON Schema with
   `$id: "https://raw.githubusercontent.com/Saber5656/mcp-medic/main/schemas/report.schema.json"`,
   title `mcp-medic report`, and writes it; CI (`build`) fails when the committed file
   drifts from generated output (`git diff --exit-code` in a `check:schema` script —
   wire into CI in this issue).
5. Exit-level: re-export `computeExitLevel` usage here as
   `reportExitCode(report): 0 | 1` for the CLI (single place mapping summary+failOn).

# Acceptance Criteria

- [ ] Assembled report from a synthetic full pipeline (planted secrets in inventory,
      including inside `raw`) validates against `schema.ts` AND the emitted
      `schemas/report.schema.json` (zod round-trip in test), with all env/header
      values masked and **no `raw` property anywhere** (negative JSON-path scan).
- [ ] Determinism: two assemblies with fixed clock → identical strings.
- [ ] `check:schema` behavior verified by executing the actual npm script: test
      copies a stale `report.schema.json` into a temp workspace, runs
      `npm run check:schema` there, asserts non-zero exit.
- [ ] Suppressed findings appear with `suppressed: true` and are excluded from
      summary counts.
- [ ] `schemaVersion` is `1` and `strict()` rejects unknown keys (negative test).

# Validation

`npm test -- --run tests/unit/report`; `npm run build` produces/validates the schema
file; reviewer diffs schema fields against DESIGN.md §4.

# Dependencies

02 (CI workflow to host the `check:schema` step), 04, 07 (redaction), 19.

# Non-goals

- No text/markdown rendering (32/33), no CLI wiring (34), no SARIF (v2).

# Design References

- DESIGN.md §4, §13; §2.3 (exit codes)
