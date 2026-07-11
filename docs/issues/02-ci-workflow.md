# Title

CI workflow: cross-platform test matrix with pinned actions

# Summary

Add a GitHub Actions workflow that lints, typechecks, tests, and builds on
{ubuntu, macos, windows} × Node {20, 22, 24} for every PR and push to `main`, with all
actions pinned by commit SHA and minimal permissions.

# Context

DESIGN.md §17.2 mandates the matrix (Windows is a first-class target because path and
process semantics differ). §14.4 mandates supply-chain hygiene for the workflow itself.

# Scope

- `.github/workflows/ci.yml`
- `.github/dependabot.yml` (npm + github-actions ecosystems, weekly)

# Detailed Requirements

1. Trigger: `pull_request` and `push` to `main`. Concurrency group per ref with
   `cancel-in-progress: true`.
2. Top-level `permissions: { contents: read }` — nothing else.
3. Job `test`: strategy matrix `os: [ubuntu-latest, macos-latest, windows-latest]` ×
   `node: [20, 22, 24]`, `fail-fast: false`.
   Steps: checkout → setup-node (with `cache: npm`) → `npm ci` →
   `npm audit --omit=dev --audit-level=high` (blocking; runtime-dependency
   vulnerabilities fail CI — DESIGN §3.2) → `npm run typecheck` →
   `npm run lint` → `npm test -- --run` → `npm run build` → `node dist/cli.js`
   (smoke-run the built artifact).
4. Every `uses:` pinned to a full 40-char commit SHA with a trailing
   `# vX.Y.Z` comment (e.g. `actions/checkout@<sha> # v4.x`). Resolve current SHAs at
   implementation time; do not use tag references.
5. Dependabot config: `package-ecosystem: npm` and `github-actions`, weekly,
   grouped minor/patch updates.
6. Windows notes: do not add `shell:` overrides that would mask path bugs; npm scripts
   must run under the default shell on all three OSes.
7. Name the workflow `CI` (the README badge added in issue 36 references it by this
   name — no badge work in this issue).

# Acceptance Criteria

- [ ] Workflow is green on all 9 matrix cells for a trivial PR.
- [ ] `actionlint` (run locally or via a one-off step) reports no errors.
- [ ] No `uses:` reference by tag or branch; all SHA-pinned with version comments.
- [ ] Workflow has no `write` permission of any kind.
- [ ] Dependabot config validates (GitHub UI shows both ecosystems).

# Validation

Open a draft PR touching a source file; confirm all 9 cells run and pass; confirm
cancellation works by pushing twice quickly.

# Dependencies

01 (scripts and lockfile must exist).

# Non-goals

- No release/publish workflow (issue 37), no CodeQL (issue 03), no coverage upload.

# Design References

- DESIGN.md §17.2 (CI matrix), §14.4 (release/workflow security)
