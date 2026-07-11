# Title

Release automation: npm publish with provenance

# Summary

Add the tag-triggered release workflow: verify green CI, build, run the full test
suite, publish `mcp-medic` to npm with provenance attestation, and create a GitHub
Release with generated notes — with the npm credential path documented as an
owner-side setup step.

# Context

DESIGN.md §14.4 and §18. Publishing is the supply-chain-critical moment for a security
tool: provenance, pinned actions, minimal permissions, and a human tag as the trigger.
The npm account/trusted-publisher configuration is owner-held (working agreement:
agents never handle credentials) — this issue ships the workflow and the runbook.

# Scope

- `.github/workflows/release.yml`
- `docs/RELEASING.md` (runbook: version bump → tag → verify)
- `CHANGELOG.md` (initialized with Unreleased section + 0.1.0 template)

# Detailed Requirements

1. Triggers: `push` of tag matching `v[0-9]+.[0-9]+.[0-9]+` (real release) **and**
   `workflow_dispatch` with boolean input `dry_run` (default `true`). Publish and
   GitHub-Release steps are guarded with
   `if: startsWith(github.ref, 'refs/tags/v') && inputs.dry_run != 'true'`-style
   conditions so a dispatch run can never publish. Job-level
   `permissions: { contents: write, id-token: write }` (id-token for provenance;
   contents for the GitHub Release) — nothing broader; no `pull_request` trigger.
2. Steps: checkout (SHA-pinned) → setup-node with `registry-url` → `npm ci` →
   `npm run typecheck && npm run lint && npm test -- --run && npm run build` →
   guard (tag runs only): tag version === package.json version (fail otherwise) →
   `npm publish --provenance --access public` (`--dry-run` appended on dispatch
   dry-runs) → `gh release create` (tag runs only) with auto-generated notes +
   `CHANGELOG.md` extract, step env `GH_TOKEN: ${{ github.token }}` (the `gh` CLI
   does not authenticate without it).
3. Credentials: prefer npm **trusted publishing** (OIDC, no token) if configured;
   fallback `NODE_AUTH_TOKEN` from repo secret `NPM_TOKEN` (granular, publish-only).
   The workflow supports both paths behind a repository variable
   `NPM_TRUSTED_PUBLISHING=true|false`; `docs/RELEASING.md` documents the owner setup
   for each (npmjs.com → package → Publishing access), explicitly marked
   **owner manual step**.
4. Post-publish verification step (**tag runs only — skipped on dry-run**, where
   nothing was published): `npm view mcp-medic@<version> dist.integrity` and
   `npm audit signatures` in a clean directory installing the just-published
   version, with up to 5 retries × 30 s backoff (registry propagation is not
   instant); failure after retries marks the workflow red (release exists but
   unverified — runbook covers response: deprecate the version, never
   unpublish-and-reuse).
5. `docs/RELEASING.md` runbook: preflight checklist (CI green on main, CHANGELOG
   updated, version bumped via `npm version --no-git-tag-version` + PR), tagging
   commands, what to do on partial failure (publish ok / release notes failed, etc.),
   yank policy (deprecate, not unpublish).
6. First release target is `0.1.0` after issue 35 is green (stated in the runbook;
   the workflow itself is version-agnostic).

# Acceptance Criteria

- [ ] Dry-run path: a `workflow_dispatch` input `dry_run=true` executes every step
      with `npm publish --dry-run` and skips the GitHub Release (no throwaway
      package names are ever published); acceptance = successful dry-run execution
      on main.
- [ ] Version-mismatch guard fails a deliberately mismatched dry-run.
- [ ] All actions SHA-pinned; `actionlint` clean; permissions exactly as specified.
- [ ] RELEASING.md walked through by the owner (checklist reviewed) before first tag.
- [ ] CHANGELOG format follows Keep a Changelog headings.

# Validation

`workflow_dispatch` dry-run green on all steps; owner sign-off on the runbook;
first real `v0.1.0` release performed by the owner tags-only (post-v1, outside this
issue's completion gate — the dry-run is the gate).

# Dependencies

02, 34 (35 green is a process gate for the real release, not for merging this issue).

# Non-goals

- No auto-versioning/semantic-release, no Homebrew tap or SEA binaries (v2), no
  npm token handling by agents (owner-only), no signing beyond npm provenance.

# Design References

- DESIGN.md §14.4, §18; ADR-002; ISSUE_PLAN owner handoff items 2–3
