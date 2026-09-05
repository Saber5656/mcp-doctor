# Title

Community & security baseline files (SECURITY.md, CodeQL, templates)

# Summary

Add the OSS hygiene set: `SECURITY.md` (private vulnerability reporting, scope,
disclosure timeline), `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, GitHub issue/PR
templates, and a CodeQL analysis workflow. (Dependabot configuration is owned by
issue 02, not this issue.)

# Context

DESIGN.md §14.4 treats release/repo security as part of the product. The project will
be public; these files must exist before the first external contribution or report.

# Scope

- `SECURITY.md`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`
- `.github/ISSUE_TEMPLATE/bug_report.yml`, `.github/ISSUE_TEMPLATE/rule_proposal.yml`,
  `.github/ISSUE_TEMPLATE/config.yml`, `.github/PULL_REQUEST_TEMPLATE.md`
- `.github/workflows/codeql.yml`

# Detailed Requirements

1. `SECURITY.md`:
   - Report privately via GitHub Security Advisories (`/security/advisories/new`).
   - Response target: acknowledge ≤ 7 days; fix-or-mitigation target 90 days.
   - In-scope examples: redaction bypass (secret emitted in any output), consent-gate
     bypass (process spawned without `--connect`), sanitization bypass (terminal
     escape reaching output), path traversal, dependency vulnerabilities.
   - The tool's security promises, stated exactly as these four testable claims
     (this wording is the single source; README copies it verbatim in issue 36):
     1. "mcp-medic never modifies MCP client configuration files; the only file it
        ever writes is the report you request with `--output`."
     2. "mcp-medic never executes a configured server command unless you pass
        `--connect` and approve the printed execution plan."
     3. "mcp-medic performs no network I/O of its own unless you pass `--online`
        (npm registry metadata; package names only) — servers probed under
        `--connect` may use the network themselves."
     4. "mcp-medic collects no telemetry and performs no update checks."
2. `CONTRIBUTING.md`: dev setup (`npm ci`, `npm test`), how docs/DESIGN.md governs
   behavior changes, adapter-addition guide pointer (fixtures + contract tests),
   conventional commit style `type(scope): subject`, PR checklist (tests + docs).
3. `CODE_OF_CONDUCT.md`: Contributor Covenant v2.1, contact = repo owner via GitHub.
4. Bug template asks for: OS, Node version, client(s), `mcp-medic clients` output,
   redacted `--format json` report. Rule-proposal template asks for: threat/defect
   description, example config snippet, proposed severity, false-positive analysis.
5. CodeQL workflow: steps `github/codeql-action/init@<sha>` with
   `languages: javascript-typescript` followed by
   `github/codeql-action/analyze@<sha>` (no autobuild step — TypeScript needs none);
   both SHA-pinned with version comments; triggers: PR + weekly schedule;
   `permissions: security-events: write, contents: read` only.

# Acceptance Criteria

- [ ] All files render correctly on GitHub (templates appear in the new-issue chooser).
- [ ] CodeQL run completes green on default branch.
- [ ] SECURITY.md lists the four security promises verbatim-testable.
- [ ] No workflow permission broader than stated.

# Validation

Verify each template renders in the new-issue chooser (preview is sufficient; if test
issues are created, close them immediately with a `test` note); trigger CodeQL via a
PR; verify the advisory link resolves on the (public or private) repo.

# Dependencies

01.

# Non-goals

- Branch-protection/ruleset changes (owner-managed), release workflow (37), README (36).

# Design References

- DESIGN.md §14 (security model), §14.4 (release security)
