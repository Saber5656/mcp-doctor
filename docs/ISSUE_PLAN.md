# mcp-medic — v1 Issue Plan

Status: active. Last updated: 2026-07-11.
Derived from `docs/DESIGN.md`. GitHub Issues are generated 1:1 from `docs/issues/*.md`;
if they ever disagree, these files win and the GitHub Issues are stale.

## v1 completion statement

When every issue listed below (01–37) is completed and its Validation section passes,
mcp-medic v1 as specified in `docs/DESIGN.md` is complete:

> A user on macOS, Linux, or Windows can run `npx mcp-medic` to statically diagnose the
> MCP configurations of Claude Desktop, Claude Code, Codex CLI, Cursor, Windsurf,
> VS Code, Zed, and Gemini CLI (plus declarative custom clients); add `--connect` to
> verify servers via real MCP handshakes under an explicit consent gate; add `--online`
> to check npm-registry facts; and consume the results as colored text, stable
> versioned JSON, or Markdown with documented exit codes — with secrets never emitted
> on any output path, and the package released to npm with provenance.

Remaining v1 work outside these issues: none, except newly discovered implementation
unknowns (§ Known unknowns), which must become new numbered issues before implementation.

## Issue list (recommended execution order)

| # | File | Title | Wave |
|---|---|---|---|
| 01 | `issues/01-repo-scaffolding.md` | Repository scaffolding: package, TypeScript, build, test, lint | 0 |
| 02 | `issues/02-ci-workflow.md` | CI workflow: cross-platform test matrix with pinned actions | 0 |
| 03 | `issues/03-community-security-files.md` | Community & security baseline files (SECURITY.md, CodeQL, Dependabot) | 0 |
| 04 | `issues/04-core-model.md` | Core domain model and types (`src/core/model.ts`) | 1 |
| 05 | `issues/05-paths-module.md` | Cross-platform path resolution module (`src/core/paths.ts`) | 1 |
| 06 | `issues/06-config-readers.md` | Tolerant JSONC/TOML readers with locations and limits | 1 |
| 07 | `issues/07-redaction-module.md` | Secret detection, masking, and output sanitization (`src/core/redact.ts`) | 1 |
| 08 | `issues/08-adapter-contract.md` | ClientAdapter contract, registry, and shared contract-test harness | 1 |
| 09 | `issues/09-adapter-claude-desktop.md` | Adapter: Claude Desktop | 2 |
| 10 | `issues/10-adapter-claude-code-user-local.md` | Adapter: Claude Code — `~/.claude.json` user & local scopes | 2 |
| 11 | `issues/11-adapter-claude-code-project-managed.md` | Adapter: Claude Code — `.mcp.json`, settings state, managed scope | 2 |
| 12 | `issues/12-adapter-codex.md` | Adapter: Codex CLI (TOML) | 2 |
| 13 | `issues/13-adapter-cursor.md` | Adapter: Cursor | 2 |
| 14 | `issues/14-adapter-windsurf.md` | Adapter: Windsurf | 2 |
| 15 | `issues/15-adapter-vscode.md` | Adapter: VS Code | 2 |
| 16 | `issues/16-adapter-zed.md` | Adapter: Zed | 2 |
| 17 | `issues/17-adapter-gemini.md` | Adapter: Gemini CLI | 2 |
| 18 | `issues/18-adapter-custom.md` | Declarative custom adapters (`--adapter <file>`) | 2 |
| 19 | `issues/19-rule-engine.md` | Rule engine, registry, ignore/fail-on plumbing | 3 |
| 20 | `issues/20-rules-integrity.md` | Integrity rules MM101–MM108 | 3 |
| 21 | `issues/21-rules-broken-static.md` | Broken-server static rules MM201–MM206 + package-spec util | 3 |
| 22 | `issues/22-rules-duplicates.md` | Duplicate & consistency rules MM301–MM304 | 3 |
| 23 | `issues/23-rules-security.md` | Security rules MM401–MM410 | 3 |
| 24 | `issues/24-rules-hygiene.md` | Hygiene rules MM501–MM505 | 3 |
| 25 | `issues/25-probe-fixtures.md` | Probe test fixtures: scripted fake MCP servers + HTTP fixtures | 4 |
| 26 | `issues/26-probe-stdio.md` | stdio probe engine with hardened spawn and failure classification | 4 |
| 27 | `issues/27-probe-http.md` | HTTP/SSE probe engine | 4 |
| 28 | `issues/28-probe-runner.md` | Probe runner: eligibility, dedupe, concurrency, MM605/MM610 | 4 |
| 29 | `issues/29-online-npm-client.md` | npm registry client + rules MM701/MM702/MM704 | 5 |
| 30 | `issues/30-online-typosquat.md` | Typosquat heuristic + curated package list (MM703) | 5 |
| 31 | `issues/31-report-json.md` | Report assembly, JSON renderer, exported JSON schema | 6 |
| 32 | `issues/32-render-text.md` | Terminal text renderer with sanitization enforcement | 6 |
| 33 | `issues/33-render-markdown.md` | Markdown renderer | 6 |
| 34 | `issues/34-cli-wiring.md` | CLI wiring: commands, flags, consent gate, orchestrator, exit codes | 6 |
| 35 | `issues/35-e2e-suite.md` | E2E golden tests + security regression suite | 7 |
| 36 | `issues/36-readme-docs.md` | README and user documentation (generated rule table) | 7 |
| 37 | `issues/37-release-automation.md` | Release automation: npm publish with provenance | 7 |

## Dependency table

`A ← B` means B depends on A. Only direct dependencies listed.

| Issue | Depends on |
|---|---|
| 01 | — |
| 02 | 01 |
| 03 | 01 |
| 04 | 01 |
| 05 | 01 |
| 06 | 01 |
| 07 | 01, 04 |
| 08 | 04, 05, 06 |
| 09, 10, 12–17 (adapters) | 08 (and nothing else; mutually independent) |
| 11 | 08, 10 (shared claude-code helpers: env expansion, entry normalization) |
| 18 | 08 |
| 19 | 04 |
| 20 | 19, 06 |
| 21 | 19, 05 |
| 22 | 19, 21 (package-spec util) |
| 23 | 19, 07, 21 (package-spec util), 05 |
| 24 | 19, 10 (`RESERVED_NAMES` import) |
| 25 | 01 |
| 26 | 04, 05, 07, 19, 25 |
| 27 | 04, 07, 19, 25 |
| 28 | 19, 22, 26, 27 |
| 29 | 19, 21 (package-spec util), 07 |
| 30 | 29 |
| 31 | 02, 04, 07, 19 |
| 32 | 31, 07 |
| 33 | 31, 07 |
| 34 | 08, 18, 19, 28, 29, 31, 32, 33 (integration point; adapters 09–17 pluggable later but required for v1 sign-off) |
| 35 | 34, 09–17, 25, 29, 30 |
| 36 | 34, 35 |
| 37 | 02, 34 |

## Implementation waves

| Wave | Issues | Parallelism |
|---|---|---|
| 0 — Foundations | 01 → 02, 03 | 02 ∥ 03 after 01 |
| 1 — Core | 04 ∥ 05 ∥ 06 → 07 → 08 | 04/05/06 parallel; 07 after 04; then 08 |
| 2 — Adapters | 09 ∥ 10 ∥ 12 ∥ 13 ∥ 14 ∥ 15 ∥ 16 ∥ 17 ∥ 18; 11 after 10 | high parallelism |
| 3 — Rules | 19 → 20 ∥ 21 ∥ 24 → 22, 23 | 22 and 23 after 21 |
| 4 — Probe | 25 → 26 ∥ 27 → 28 | |
| 5 — Online | 29 → 30 | can run parallel to wave 4 |
| 6 — Output & CLI | 31 → 32 ∥ 33 → 34 | |
| 7 — Ship | 35 → 36; 37 ∥ 35 | 35 green gates the first real *release event* (issue 37's own merge gate is its dry-run) |

Waves 2, 3, 4/5 offer wide parallel lanes for multiple implementation agents; each
issue is designed to touch a disjoint file set (adapters especially).

## Coverage table: DESIGN.md § → issues

| DESIGN.md section | Covered by |
|---|---|
| §1 Product definition, goals/non-goals | all (framing); 36 (user-facing statement) |
| §2 CLI contract, exit codes, consent flow | 34 (+31 exit-code computation, 35 verification) |
| §3 Architecture, layout, dependencies | 01, 04 (+02 npm-audit/Dependabot, 03 policy docs) |
| §4 Core data model | 04 |
| §5 Orchestrator pipeline | 34 |
| §6 Path resolution | 05 |
| §7.1 Adapter contract | 08 |
| §7.2 Built-in adapters | 09, 10, 11, 12, 13, 14, 15, 16, 17 |
| §7.3 Transport normalization | 04 (single-owner helper; adapters record flags only) |
| §7.4 Custom adapters | 18 |
| §8 Parsing layer | 06 |
| §9 Rule engine | 19 |
| §10 MM1xx | 20 |
| §10 MM2xx | 21 |
| §10 MM3xx | 22 |
| §10 MM4xx | 23 |
| §10 MM5xx | 24 (MM503/505 inputs from 10, 11) |
| §10 MM6xx | 26, 27, 28 |
| §10 MM7xx | 29, 30 |
| §11.1–11.4 Probe engine | 25, 26, 27, 28 |
| §11.5 Registry client | 29 |
| §12 Redaction & sanitization | 07 (+32 enforcement, 35 guarantee test) |
| §13 Reporting | 31, 32, 33 |
| §14 Security model | 03, 07, 23, 26, 34, 35, 37 |
| §15 Failure modes | 34, 35 |
| §16 Performance targets | 35 |
| §17 Testing strategy | 08 (harness), 25, 35, 02 (matrix) |
| §18 Distribution & release | 01, 36, 37 |
| §19 v2 deferred | not in v1 (tracked below) |
| §20 Known unknowns | tracked below |

Every normative DESIGN.md section maps to at least one issue; no v1 behavior lives
only in prose.

## Whole-product validation strategy

1. **Per-issue**: each issue's Validation section is mandatory for closure (unit /
   contract / integration tests named per issue).
2. **Continuous**: CI matrix (issue 02) — 3 OS × Node 20/22/24 on every PR.
3. **System acceptance** (issue 35): fake-home fixture corpus with ≥1 planted defect
   per rule family and an `all-healthy` home. Required outcomes: zero false negatives
   on planted defects, zero findings on healthy home, zero secret leakage across all
   output modes, exit codes as documented.
4. **Security regression** (issue 35): consent-gate behavior, spawn hardening,
   kill-escalation orphan check, redaction guarantee, offline-mode no-network assert.
5. **Release gate** (issue 37): publish only from green CI on a tag; provenance
   attestation verified post-publish (`npm audit signatures`).

## Deferred to v2 (not planned as issues)

From DESIGN.md §19: `fix` write mode; client log analysis; SARIF; server sync/migrate;
more clients (JetBrains, LM Studio, Cline…); `.mcpb`/connector visibility; i18n;
`--only` + rc-file config; watch mode; PyPI/Docker registry sources.

## Known unknowns (may create additional issues)

From DESIGN.md §20: U1 managed-mcp.json paths (→ issue 11 carries a verification task);
U2 VS Code profiles/Insiders (→ 15); U3 Windsurf `serverUrl` precedence (→ 14);
U4 Zed extension entry shapes (→ 16); U5 Codex project trust gating (→ 12);
U6 Windows kill-tree reliability (→ 26/35); U7 ancient-protocol servers (→ 26);
U8 Gemini enablement schema (→ 17).
Resolution rule: if an unknown invalidates a documented fact, update
`docs/research/client-config-formats.md` and the affected issue file **before** coding.

## Owner handoff items (manual, not implementation issues)

1. Rename GitHub repo `Saber5656/mcp-doctor` → `mcp-medic`; rename local dir; update
   remotes (ADR-001).
2. Reserve the npm name `mcp-medic` (placeholder publish or org scope) — credentials
   are owner-held per working agreement.
3. Configure npm trusted publishing (or `NPM_TOKEN` secret) for issue 37.
4. Repo settings already in place: main-branch ruleset (no direct push). Verify Actions
   permissions when issue 02 lands.
