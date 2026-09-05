# Title

README and user documentation (generated rule table)

# Summary

Write the public README (install, quickstart, consent model, CI recipe, output
formats, exit codes) plus `docs/rules.md` generated from the rule registry, replacing
the placeholder README and wiring a docs-generation script that CI keeps honest.

# Context

DESIGN.md §18 (README contents) and the single-source rule registry (issue 19's
`ruleTable()`). The README is the front door of the OSS launch; the consent model
(ADR-003) and privacy stance (ADR-004) are headline content, not fine print.

# Scope

- `README.md` (English primary; short Japanese section `## 日本語` with a 5-line
  summary — per ADR-007 docs may be bilingual)
- `docs/rules.md` (generated) + `scripts/gen-rules-doc.ts` + `npm run gen:docs`
- CI check `check:docs` (regenerate + `git diff --exit-code`)

# Detailed Requirements

1. README sections in order:
   - Badges: CI, npm version, license.
   - One-paragraph pitch + 30-second usage block
     (`npx mcp-medic` → `npx mcp-medic --connect` → `--online`).
   - Example output screenshot-as-text block (from an e2e golden, hand-trimmed).
   - Supported clients table (8 clients + custom adapters). Required columns:
     Client | Scopes scanned | Formats | Known limitations. The limitations column
     is sourced from the research doc's cross-client observations (claude.ai
     connectors invisible; VS Code non-default profiles unscanned; UI-managed
     enable state reported as `unknown`) — the README must not overclaim coverage.
   - "What it checks" — category overview linking `docs/rules.md`.
   - **Security & privacy** section: copies the **four** testable claims from
     SECURITY.md verbatim (issue 03 defines them: never modifies configs; never
     executes without `--connect` + approval; no own network I/O without
     `--online`; no telemetry), adds the redaction promise, links SECURITY.md.
     SECURITY.md is the single wording source.
   - CI recipe: GitHub Actions snippet with `--fail-on warn --format json`, artifact
     upload, and `$GITHUB_STEP_SUMMARY` markdown example.
   - Exit codes table (must equal DESIGN §2.3).
   - Flags reference (generated from commander `--help` output, committed).
   - Roadmap (v2 items from DESIGN §19, abbreviated), Contributing, License.
2. `scripts/gen-rules-doc.ts`: imports `ruleTable` from the built `dist/index.js`
   (issue 34 re-exports it from `src/index.ts`), emits `docs/rules.md` — one section
   per category, table `ID | Severity | Applies to | Clients | Summary`, plus
   per-rule remediation notes where the registry provides them. `gen:docs` script =
   `npm run build && node --experimental-strip-types scripts/gen-rules-doc.ts`
   (or a plain compiled-js script — implementer's choice, documented in
   package.json).
3. `check:docs` in CI: run generator, fail on diff (same pattern as `check:schema`).
4. README claims audit: every behavioral claim must cite an existing test (reviewer
   checklist embedded as an HTML comment at the top of README listing claim → test
   file, kept updatable).

# Acceptance Criteria

- [ ] `npm run gen:docs` idempotent; CI check wired and green.
- [ ] README quickstart commands copy-paste-run successfully against the built
      package (`npm pack` + `npx ./mcp-medic-*.tgz`) on a fixture home — scripted as
      a test in `tests/e2e/readme-quickstart.test.ts`.
- [ ] Exit-code table equals DESIGN §2.3 (reviewer diff); flags reference equals
      normalized `mcp-medic --help` output (snapshot equality — exit codes are not
      part of commander help).
- [ ] The four security claims byte-match their SECURITY.md counterparts.
- [ ] Japanese section present, ≤ 10 lines, content-consistent with the pitch.

# Validation

Docs CI check + readme-quickstart e2e test; human review of tone/clarity by the
repository owner before release.

# Dependencies

34, 35 (hard: the README example block derives from a 35 golden, and
`tests/e2e/readme-quickstart.test.ts` extends the 35 e2e harness).

# Non-goals

- No docs website, no logo/branding work, no localization beyond the short Japanese
  summary, no CHANGELOG backfill (starts at first release, issue 37).

# Design References

- DESIGN.md §18; ADR-003/004/005/007; ISSUE_PLAN handoffs (npm name note)
