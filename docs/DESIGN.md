# mcp-medic — Design Document (v1)

Status: approved design for v1 implementation. Last updated: 2026-07-11.
Canonical source of truth for requirements and architecture. Issues in `docs/issues/`
are derived from this document via `docs/ISSUE_PLAN.md`.

Related documents:
- `docs/research/client-config-formats.md` — verified per-client config facts (normative for adapters)
- `docs/research/prior-art.md` — competitor analysis and differentiation
- `docs/decisions/ADR-*.md` — recorded decisions

> Naming note: the repository is currently named `mcp-doctor` and will be renamed to
> **`mcp-medic`** (ADR-001). All code, docs, and the npm package use `mcp-medic`.

---

## 1. Product definition

**mcp-medic** is a read-only, cross-client health-check CLI for MCP (Model Context
Protocol) configurations. It scans the MCP server definitions of every supported AI
client installed on a machine, normalizes them into one inventory, and diagnoses:

1. **Broken servers** — unparseable configs, missing commands/runtimes, malformed
   entries, and (opt-in) servers that fail to start or complete an MCP handshake.
2. **Duplicates & inconsistencies** — the same server defined in multiple scopes or
   clients, shadowed definitions, divergent env/version pins.
3. **Security risks** — plaintext secrets, loose file permissions, non-TLS endpoints,
   unpinned package execution, secrets in argv/URLs, shell-wrapper commands,
   trust-bypass flags, deprecated transports.

One command, actionable findings, safe defaults:

```
npx mcp-medic            # static scan of all detected clients
npx mcp-medic --connect  # additionally start each server and verify the MCP handshake
npx mcp-medic --online   # additionally check npm registry facts (opt-in network)
```

### 1.1 Target users

| Persona | Need |
|---|---|
| Multi-client MCP user (primary) | "Why is this server broken in Cursor but fine in Claude Code?" / "What did I leave configured?" |
| Security-conscious engineer | "Are any of my configs leaking tokens or executing unpinned code?" |
| Team lead / CI | Machine-readable report + exit codes to gate a repo's `.mcp.json` in CI |

### 1.2 Design principles

1. **Read-only** (v1): never writes to any client config file. No fix mode (ADR-005).
2. **Secure by default**: no process execution and no network without explicit flags
   (`--connect`, `--online`) (ADR-003, ADR-004).
3. **Never leak**: secrets are redacted on every output path; untrusted strings are
   sanitized before terminal rendering.
4. **Deterministic & CI-friendly**: stable JSON schema, documented exit codes,
   `--no-color`, no interactive prompts outside TTY.
5. **Facts over guesses**: every diagnostic cites file + location + remediation; rules
   with false-positive risk are `warn`/`info`, never `error`.

### 1.3 v1 goals (must ship)

- Static scan across 8 clients: Claude Desktop, Claude Code, Codex CLI, Cursor,
  Windsurf, VS Code, Zed, Gemini CLI — plus a declarative custom-adapter mechanism.
- Dynamic probe (`--connect`): stdio spawn + MCP handshake + `tools/list`; HTTP(S)
  endpoint probe; failure classification.
- Online registry checks (`--online`): package existence, deprecation, typosquat
  similarity, outdated pins.
- Rule catalog (§10) with severities, per-rule remediation, `--ignore`, `--fail-on`.
- Renderers: terminal text, JSON (versioned schema), Markdown.
- Subcommands: `scan` (default), `list`, `clients`, `rules`.
- Cross-platform: macOS, Linux, Windows. Node ≥ 20.
- OSS hygiene: MIT license, SECURITY.md, CI matrix, npm provenance releases.

### 1.4 v1 non-goals

- No `fix`/write mode of any kind (v2; ADR-005).
- No TUI/interactive browsing, no watch mode.
- No i18n; CLI output is English only (ADR-007). Repo README may be bilingual.
- No client log-file analysis (Claude Desktop `mcp*.log` etc.) — v2.
- No inspection of claude.ai connectors, Claude Desktop extension store, or other
  cloud-managed servers invisible to the filesystem (explicitly disclosed in output).
- No MCP server code linting / tool-schema quality analysis (mcp-checkup's niche).
- No telemetry, no auto-update, no crash reporting.
- No plugin system with executable adapters (declarative custom adapters only).

---

## 2. CLI contract

Binary name: `mcp-medic` (npm package `mcp-medic`, `bin` entry `mcp-medic`).

### 2.1 Commands

| Command | Purpose |
|---|---|
| `mcp-medic [scan]` | Full diagnostic scan (default command) |
| `mcp-medic list` | Print normalized server inventory only (no rules) |
| `mcp-medic clients` | Show supported clients, detection status, config file paths found |
| `mcp-medic rules` | Print the rule catalog (id, severity, category, summary) |

### 2.2 `scan` options

| Flag | Type / default | Meaning |
|---|---|---|
| `--connect` | bool, off | Enable dynamic probing. Spawns stdio servers / contacts configured URLs. Requires confirmation (§2.4) |
| `--online` | bool, off | Enable npm-registry lookups (package names only are sent) |
| `--yes` | bool, off | Skip the `--connect` confirmation (required for `--connect` in non-TTY) |
| `--client <id>` | repeatable | Restrict to specific client ids (`claude-desktop`, `claude-code`, `codex`, `cursor`, `windsurf`, `vscode`, `zed`, `gemini-cli`, custom ids) |
| `--project <dir>` | default: cwd | Project root used to locate project-scope config files |
| `--adapter <file>` | repeatable | Load a declarative custom adapter definition (§7.4) |
| `--format <fmt>` | `text` \| `json` \| `markdown`, default `text` | Output format |
| `--output <file>` | default: stdout | Write report to file |
| `--fail-on <level>` | `error` \| `warn` \| `info` \| `never`, default `error` | Threshold for exit code 1 |
| `--ignore <ruleId>` | repeatable | Suppress findings of a rule (recorded in report as suppressed) |
| `--timeout <ms>` | default `15000` | Per-server probe startup timeout |
| `--concurrency <n>` | default `4`, max `16` | Parallel probes |
| `--no-color` | bool | Disable ANSI colors (also honors `NO_COLOR` env) |
| `--quiet` | bool | Findings only, no banner/progress |
| `--verbose` | bool | Extra diagnostics to stderr |

`list`, `clients`, `rules` accept `--format`, `--output`, `--no-color` where meaningful.

### 2.3 Exit codes

| Code | Meaning |
|---|---|
| 0 | Scan completed; no findings at or above `--fail-on` level |
| 1 | Scan completed; at least one finding at/above `--fail-on` |
| 2 | Usage/configuration error (unknown flag, invalid adapter file, `--connect` without `--yes` in non-TTY) |
| 3 | Internal error (unexpected exception; stack to stderr with `--verbose`) |
| 130 | Interrupted (SIGINT): children killed, partial report emitted with an `interrupted` note |

Probe failures of *target servers* are findings (exit 0/1 semantics), never exit 2/3.

### 2.4 `--connect` consent flow

1. Static scan runs first and builds the inventory.
2. The CLI prints the exact execution plan: for each probe-eligible server, the literal
   `command` + `args` (redacted env keys listed by name only) or the URL to contact.
3. TTY: single `Proceed? [y/N]` prompt (default No). Non-TTY: refuse with exit 2 unless
   `--yes` was given. `--yes` in TTY skips the prompt.
4. Servers marked disabled (`enabled: false`) are never probed.
5. `headersHelper`-style dynamic commands are **never executed** by mcp-medic; the HTTP
   probe runs without those headers and classifies an auth rejection as MM607 (info).

---

## 3. Architecture overview

```
                ┌───────────────────────────────────────────────┐
                │                   CLI layer                   │
                │   commander wiring · flags · exit codes       │
                └───────────────┬───────────────────────────────┘
                                │ ScanOptions
                ┌───────────────▼───────────────┐
                │        Orchestrator           │  src/core/orchestrator.ts
                │  discover → parse → normalize │
                │  → rules → (probe) → (online) │
                │  → report                     │
                └──┬─────────┬─────────┬────────┘
                   │         │         │
        ┌──────────▼──┐ ┌────▼─────┐ ┌─▼──────────────┐
        │ Discovery   │ │ Rule     │ │ Probe engine   │
        │ (adapters)  │ │ engine   │ │ stdio / http   │
        └──────────┬──┘ └────┬─────┘ └─┬──────────────┘
                   │         │         │
        ┌──────────▼─────────▼─────────▼──────────────┐
        │ Core: model · paths · parsers · redaction   │
        └──────────────────────┬──────────────────────┘
                               │ Report
                ┌──────────────▼──────────────┐
                │ Renderers: text/json/md     │
                └─────────────────────────────┘
```

Data flow is one-directional; every stage is a pure function over immutable inputs
except the probe engine (process/network side effects, gated).

### 3.1 Module and file layout

```
mcp-medic/
├── package.json              # name mcp-medic, type: module, bin, engines >=20
├── tsconfig.json             # strict, NodeNext
├── tsup.config.ts            # bundle: dist/cli.js (bin), dist/index.js (lib)
├── vitest.config.ts
├── eslint.config.js          # flat config + typescript-eslint
├── LICENSE                   # MIT
├── SECURITY.md · CONTRIBUTING.md · CODE_OF_CONDUCT.md
├── schemas/
│   └── report.schema.json    # generated from zod (build step), committed
├── src/
│   ├── index.ts              # programmatic API: scan(options): Promise<Report>
│   ├── cli/
│   │   ├── main.ts           # bin entry; command registration; exit-code mapping
│   │   └── commands/{scan,list,clients,rules}.ts
│   ├── core/
│   │   ├── model.ts          # all shared types (§4)
│   │   ├── orchestrator.ts   # pipeline described above
│   │   ├── paths.ts          # cross-platform path resolution (§6)
│   │   └── redact.ts         # secret detection, masking, output sanitization (§12)
│   ├── parsers/
│   │   ├── jsonc.ts          # tolerant JSON/JSONC + line/col locations
│   │   ├── toml.ts           # smol-toml wrapper + location mapping
│   │   └── limits.ts         # file-size caps, read guards
│   ├── discovery/
│   │   ├── adapter.ts        # ClientAdapter contract + registry (§7)
│   │   ├── custom.ts         # declarative custom adapter loader (§7.4)
│   │   └── adapters/{claude-desktop,claude-code,codex,cursor,windsurf,vscode,zed,gemini}.ts
│   ├── rules/
│   │   ├── engine.ts         # evaluation, severity, ignore/fail-on (§9)
│   │   ├── registry.ts       # rule metadata table (drives `rules` cmd)
│   │   └── {integrity,broken,duplicates,security,hygiene}/*.ts
│   ├── probe/
│   │   ├── runner.ts         # consent gate, concurrency, timeouts (§11)
│   │   ├── stdio.ts          # spawn hardening + SDK handshake
│   │   ├── http.ts           # streamable HTTP / legacy SSE probe
│   │   └── classify.ts       # failure → rule mapping
│   ├── online/
│   │   ├── npm.ts            # registry metadata client (§11.5)
│   │   └── typosquat.ts      # similarity heuristics + known-package list
│   └── report/
│       ├── report.ts         # Report assembly + summary + exit-code decision
│       ├── schema.ts         # zod schema for the JSON report
│       └── render/{text,json,markdown}.ts
└── tests/
    ├── unit/**               # colocated by module
    ├── fixtures/
    │   ├── homes/<case>/     # fake $HOME trees per scenario (per-client fixtures)
    │   └── servers/*.mjs     # scripted fake MCP servers (§17.3)
    └── e2e/**                # CLI-level golden tests
```

### 3.2 Dependencies (pinned intent)

Runtime: `@modelcontextprotocol/sdk` ^1.29, `commander` ^15, `zod` ^4.4,
`smol-toml` ^1.7, `jsonc-parser` ^3.3, `picocolors` ^1.1.
Dev: `typescript` ^5, `tsup` ^8.5, `vitest` ^4.1, `eslint` ^9 + typescript-eslint,
`prettier` ^3.

Dependency policy: no other runtime deps without an ADR; no packages with postinstall
scripts; lockfile committed; Dependabot + `npm audit` in CI.

---

## 4. Core data model (`src/core/model.ts`)

```ts
export type ClientId =
  | 'claude-desktop' | 'claude-code' | 'codex' | 'cursor'
  | 'windsurf' | 'vscode' | 'zed' | 'gemini-cli'
  | (string & {});                       // custom adapters

export type Scope = 'user' | 'project' | 'local' | 'managed';
export type Transport = 'stdio' | 'http' | 'sse' | 'ws' | 'unknown';
export type Severity = 'error' | 'warn' | 'info';

export interface SourceLocation {
  file: string;                          // absolute path
  pointer?: string;                      // JSON Pointer or TOML key path, e.g. /mcpServers/github
  line?: number; column?: number;        // 1-based, when parser provides it
}

export interface ServerEntry {
  key: string;                           // stable id: `${client}:${scope}:${name}` (+ `#n` on collision)
  client: ClientId;
  scope: Scope;
  name: string;
  transport: Transport;
  command?: string;                      // stdio
  args: string[];
  env: Record<string, string>;           // values retained in memory, redacted on output
  url?: string;                          // http/sse/ws (normalized from url/serverUrl/httpUrl)
  headers: Record<string, string>;
  enabled: boolean | 'unknown';
  source: SourceLocation;
  clientSpecific: Record<string, unknown>; // headersHelper, envFile, trust, timeout, oauth, …
  raw: unknown;                          // original entry (never printed unredacted)
}

export interface ConfigFileInfo {
  client: ClientId; scope: Scope;
  path: string; exists: boolean;
  parseError?: { message: string; line?: number; column?: number };
  loadError?: { kind: 'unreadable' | 'too-large' | 'io-error';
                errno?: string; sizeBytes?: number };   // MM107 input
  structuralIssues?: Array<{ pointer: string; line?: number; column?: number;
                             kind: string; detail: string }>; // MM102 input
  duplicateKeys?: Array<{ pointer: string; key: string;
                          line: number; column: number }>;    // MM108 input
  posixMode?: number;                    // e.g. 0o600; undefined on Windows
  sizeBytes?: number;
}

export interface Inventory {
  platform: { os: NodeJS.Platform; arch: string; node: string };
  generatedAt: string;                   // ISO 8601
  projectDir: string;
  configFiles: ConfigFileInfo[];
  servers: ServerEntry[];
  visibilityNotes: string[];             // e.g. "claude.ai connectors are not visible to file scans"
}

export interface Finding {
  rule: string;                          // e.g. "MM401"
  severity: Severity;
  title: string;                         // one line, English
  detail: string;                        // specifics with redacted evidence
  remediation: string;                   // concrete next step, copy-pasteable when possible
  serverKey?: string;
  location?: SourceLocation;
  suppressed?: boolean;                  // true when matched by --ignore
}

export interface ProbeResult {
  serverKey: string;
  status: 'ok' | 'spawn-failed' | 'timeout' | 'handshake-error'
        | 'exited' | 'http-error' | 'auth-required' | 'skipped';
  startupMs?: number;
  protocolVersion?: string;
  toolCount?: number;
  toolNames?: string[];
  toolsCapability?: boolean;             // server advertised tools capability
  facts?: {                              // classification facts consumed by MM6xx rules
    errno?: string; exitCode?: number; signal?: string;
    stderrTail?: string;                 // redacted + sanitized
    stdoutNoise?: boolean;               // MM608
    httpClass?: 'dns' | 'connect' | 'timeout' | 'tls' | 'not-found'
              | 'server-error' | 'redirect';
    authRejected?: boolean;              // configured header rejected (401/403)
    fallbackTransport?: 'sse';           // legacy fallback succeeded
  };
  detail?: string;                       // redacted, sanitized
}

export interface Report {
  schemaVersion: 1;
  tool: { name: 'mcp-medic'; version: string };
  run: { timestamp: string; durationMs: number;
         mode: { connect: boolean; online: boolean };
         platform: Inventory['platform']; failOn: string };
  inventory: Inventory;                  // servers redacted via redactEntryForOutput AND with `raw` removed
  probes: ProbeResult[];
  findings: Finding[];
  summary: { errors: number; warnings: number; infos: number;
             suppressed: number; serversScanned: number; serversProbed: number };
}
```

Invariants:
- `Report` is the single serialization boundary; renderers never reach around it.
- Anything typed `string` that originated in a config file or process output passes
  through `redact.ts` sanitization before rendering (§12).
- `key` collisions (same client+scope+name twice in one file) get `#2`, `#3` suffixes
  and raise MM108.

---

## 5. Orchestrator pipeline

`scan(options)` executes:

1. **Resolve clients**: registry of built-in adapters + `--adapter` files; filter by `--client`.
2. **Discover**: each adapter returns candidate `ConfigFileInfo[]` (existence, size, mode).
3. **Parse & normalize**: adapters parse existing files → `ServerEntry[]` + parse errors.
4. **Static rules**: rule engine evaluates per-entry, per-file, and cross-inventory rules.
5. **Probe** (iff `--connect` and consent passed): probe runner executes §11; classifier
   emits probe findings.
6. **Online** (iff `--online`): npm client fetches metadata for npx/npm-resolvable
   packages; online rules run.
7. **Assemble** `Report`, compute summary + exit code, render via `--format`.

Failures in steps 2–6 for one client/server never abort the run; they degrade to
findings or `visibilityNotes`. Only invalid CLI input (exit 2) or unexpected exceptions
(exit 3) abort.

---

## 6. Path resolution (`src/core/paths.ts`)

Single module owning every filesystem location; adapters must not call `os.homedir()`
directly.

| Helper | Source of truth |
|---|---|
| `home()` | `MCP_MEDIC_HOME` (tests) → `HOME` → `USERPROFILE` |
| `appData()` | Windows `APPDATA`; mac `~/Library/Application Support`; linux `XDG_CONFIG_HOME` → `~/.config` |
| `xdgConfig()` | `XDG_CONFIG_HOME` → `~/.config` (all OSes — Zed uses this on macOS too) |
| `expandTilde(p)` | `~/` prefix expansion against `home()` |

Rules: `home()`/`appData()`/`xdgConfig()` always return absolute paths (relative env
values are ignored with fall-through); symlink handling (`fs.realpath` for display,
declared path preserved in `SourceLocation.file`) is implemented in the discovery
layer, not here; no path outside the user profile is ever *written* (v1 writes
nothing).

Test override contract: setting `MCP_MEDIC_HOME`, `APPDATA`, `XDG_CONFIG_HOME` redirects
all discovery — this is the foundation of the fake-home E2E fixtures.

---

## 7. Discovery layer

### 7.1 Adapter contract (`src/discovery/adapter.ts`)

```ts
export interface ClientAdapter {
  id: ClientId;
  displayName: string;
  /** Candidate config files for this platform (existence not required). */
  candidateFiles(ctx: DiscoveryContext): CandidateFile[];
  /** Parse one existing file into entries; never throws. */
  parseFile(file: LoadedFile, ctx: DiscoveryContext): AdapterParseResult;
  /** Optional client-specific notes (visibility caveats etc.). */
  notes?(ctx: DiscoveryContext): string[];
}

export interface CandidateFile { path: string; scope: Scope; format: 'json' | 'jsonc' | 'toml';
  origin: 'builtin' | 'custom'; }        // custom = declared via --adapter (issue 18)
export interface LoadedFile extends CandidateFile { text: string; posixMode?: number; sizeBytes: number; }
export interface AdapterParseResult {
  entries: ServerEntry[];
  parseError?: ConfigFileInfo['parseError'];
  structuralIssues?: ConfigFileInfo['structuralIssues'];
  notes?: string[];                      // parse-time visibility notes (merged into discovery notes)
}
export interface DiscoveryContext {
  platform: NodeJS.Platform;
  projectDir: string;
  env: NodeJS.ProcessEnv;                // injected for testability
}
```

**Adapters record facts; rules emit findings.** No adapter constructs a `Finding`:
parse failures land in `parseError` (→ MM101), structural violations in
`structuralIssues` (→ MM102), and every other diagnosable condition is a recorded
fact in `clientSpecific` consumed by the rule catalog. This keeps severities,
wording, and suppression in one layer.

Adapter obligations (enforced by a shared contract test harness):
1. Missing file / missing top-level key ⇒ zero entries, zero recorded issues (healthy).
2. Malformed file ⇒ `parseError` + no throw.
3. Every entry carries a `SourceLocation` with an RFC 6901-escaped JSON Pointer
   (`~` → `~0`, then `/` → `~1`; server names and project paths may contain both)
   and line/col when the parser provides it.
4. Env-var expansion follows the client's own semantics (documented per adapter in
   `docs/research/client-config-formats.md`); unresolved expansion produces the raw
   string plus a flag in `clientSpecific.unresolvedVars: string[]` (consumed by MM105).
5. Normalization table (`url`/`serverUrl`/`httpUrl` → `url` + `transport`) matches §7.3.
6. Secret material living outside the standard `env`/`headers` fields (e.g. Cursor
   `auth.CLIENT_SECRET`) is exposed via
   `clientSpecific.secretCandidates: Array<{key, value, pointer}>` — the generic
   bucket scanned by MM401 and the redaction layer.

### 7.2 Built-in adapters

Normative facts (paths, keys, fields, quirks) live in
`docs/research/client-config-formats.md`. Each adapter issue restates the exact table
it implements. Adapter-specific recorded rule inputs (facts, not findings — the MM
ids name the rules that consume them):

| Adapter | Recorded rule inputs |
|---|---|
| claude-code | `urlWithoutType` (MM103), `unresolvedVars` (MM105), name vs reserved list (MM503); reads enable/disable arrays from settings files; local scope from `~/.claude.json` `projects.*`; `projectPathExists` for stale `projects` entries (MM505) |
| codex | TOML parse error ⇒ whole-file MM101 with "Codex refuses to start on invalid TOML" detail; reads `enabled`, `startup_timeout_sec`, `bearer_token_env_var` into `clientSpecific` |
| vscode | `inputs` references (`${input:id}`) recorded in `clientSpecific.inputRefs`; never treated as secrets; `servers` key (not `mcpServers`) |
| windsurf | accepts `serverUrl` and `url`; `${env:VAR}` / `${file:...}` interpolation recorded, `${file:...}` targets never read |
| zed | accepts flat `command: string` and legacy `command: {path, args}`; `source: extension` / unrecognized `source` entries are `inventoryOnly` (excluded from probes and MM106/MM2xx) |
| gemini-cli | transport by key (`command`/`url`/`httpUrl`); reads `~/.gemini/mcp-server-enablement.json` for `enabled`; `trust: true` → MM409 input |
| claude-desktop | file may lack `mcpServers` (healthy); visibility note about UI connectors/extensions |
| cursor | `auth.CLIENT_SECRET` handled as secret material; `envFile` recorded |

### 7.3 Transport normalization

| Raw signal | Transport |
|---|---|
| `type: "stdio"` or `command` present without url-ish key | `stdio` |
| `type: "http"` / `"streamable-http"` / Gemini `httpUrl` / Codex `url` | `http` |
| `type: "sse"` / Gemini `url` | `sse` |
| `type: "ws"` (Claude Code) | `ws` |
| `url`/`serverUrl` present, no type, client ∈ {cursor, windsurf} | `http` (documented client behavior; adapter records `clientSpecific.transportAssumed`) |
| `url` present, no type, other clients | `unknown` (+ MM103 for claude-code; MM104 fires only when a declared `type` value is unrecognized) |

### 7.4 Declarative custom adapters (`--adapter <file>`)

JSON file validated by zod:

```json
{
  "id": "my-client",
  "displayName": "My Client",
  "configFiles": [{
    "path": "~/.myclient/mcp.json",
    "scope": "user",
    "format": "json",
    "serversPointer": "/mcpServers",
    "fieldMap": { "command": "command", "args": "args", "env": "env",
                   "url": "url", "headers": "headers", "type": "type" }
  }]
}
```

Constraints: `format` ∈ {json, jsonc, toml}; `serversPointer` is a JSON Pointer to an
object of name→entry for json/jsonc, or a TOML dot-path (e.g. `mcp_servers`) for
toml — emitted `source.pointer` values are always JSON Pointers over the parsed
value; `fieldMap` values are property names inside each entry; `path` supports `~`
and `${ENV}` expansion (project-scope relative paths resolve against the project
dir). Custom adapters get generic rules only (no client-specific rules). Invalid
adapter file ⇒ exit 2 with a zod-formatted message.

---

## 8. Parsing layer

- **JSON/JSONC**: `jsonc-parser` `parseTree` for all `.json` sources (VS Code and Zed
  require JSONC; tolerant parsing elsewhere prevents false positives). Provides
  node offsets → line/col; detects duplicate object keys (MM108 input).
  Strictness note recorded per client: for clients whose own parser is strict JSON
  (Claude Desktop/Code, Codex is TOML), a config that only parses as JSONC (comments,
  trailing commas) is itself a finding (MM102 detail variant "client requires strict JSON").
- **TOML**: `smol-toml`. Parse errors carry line/col. Duplicate table = parse error.
- **Limits** (`parsers/limits.ts`): max file size 5 MiB (larger ⇒ MM107 variant
  "file too large, skipped"); refuse to read through symlinks that escape the user
  profile only for *candidate* files discovered implicitly (explicit `--adapter` paths
  are read as given); read with `utf-8`, BOM tolerated.

---

## 9. Rule engine (`src/rules/engine.ts`)

```ts
export interface Rule {
  id: string;                 // "MM401"
  category: 'integrity' | 'broken' | 'duplicates' | 'security' | 'hygiene' | 'probe' | 'online';
  defaultSeverity: Severity;
  appliesTo: 'entry' | 'file' | 'inventory' | 'probe' | 'online';
  clients?: ClientId[];       // undefined = all
  summary: string;            // one line for `rules` command
  evaluate(input: RuleInput): Finding[];  // pure; no I/O except declared fs probes
}
```

- Execution order: file rules → entry rules → inventory rules → probe rules → online
  rules. Deterministic ordering: findings sorted by (severity desc, rule id, file, line).
- `--ignore MMnnn` marks matching findings `suppressed: true` (kept in JSON report,
  hidden from text output summary counts by default; `--verbose` shows them).
- `--fail-on` maps summary → exit code (§2.3).
- Rules that need filesystem probes (command existence) declare them; the engine
  provides a memoized `which()`/`stat()` service so repeated lookups are cached per run.
- Adapters never emit findings (§7.1); MM101/MM102 convert `parseError`/
  `structuralIssues` data into findings. Engine de-duplicates by
  `(rule, serverKey, location.file, location.pointer)`.

---

## 10. Rule catalog (v1 — normative)

Severities are defaults; `error` rules must have near-zero false-positive rates.

### Integrity (MM1xx)

| ID | Sev | Applies | Logic (summary) |
|---|---|---|---|
| MM101 | error | file | Config file exists but fails to parse (JSON/JSONC/TOML). Include parser message + line/col. Codex variant notes session-blocking impact |
| MM102 | error | file | Parses but violates structure: top-level servers key is not an object; entry not an object; `command`/`args`/`env`/`headers` wrong types. Also strict-JSON clients whose file only parses as JSONC |
| MM103 | error | entry (claude-code) | `url` present but `type` missing — Claude Code treats as stdio and skips the server |
| MM104 | error | entry | Unknown/unsupported `type` value for the client |
| MM105 | error | entry | Unresolved env expansion without default (client-specific syntax); names the variable(s) |
| MM106 | error | entry | Neither a runnable `command` nor a URL-ish field present |
| MM107 | warn | file | Config file unreadable (EACCES) or exceeds size cap — skipped, coverage incomplete |
| MM108 | warn | file | Duplicate server name within one file (JSON duplicate keys; last one wins silently) |

### Broken server, static (MM2xx)

| ID | Sev | Applies | Logic |
|---|---|---|---|
| MM201 | error | entry(stdio) | `command` not found (absolute path missing, or bare name absent from `PATH`) and the command is **not** a known runtime/runner — mutually exclusive with MM204 |
| MM202 | error | entry(stdio) | `command` exists but is not executable (POSIX x-bit) |
| MM203 | warn | entry(stdio) | Relative `command` or relative path in `args` — client cwd is undefined; Claude Desktop requires absolute paths |
| MM204 | error | entry(stdio) | Missing command **is** a known runtime/runner (`npx`, `bunx`, `uvx`, `pipx`, `docker`, `node`, `python`, `python3`, `deno`, `bun`) — same detection as MM201 but with runtime-installation remediation ("Node.js is not installed…") |
| MM205 | warn | entry(stdio) | Absolute path among `args` (or `envFile`) does not exist |
| MM206 | error | entry(remote) | `url`/`serverUrl`/`httpUrl` is not a syntactically valid http(s)/ws(s) URL |

### Duplicates & consistency (MM3xx) — inventory scope

| ID | Sev | Logic |
|---|---|---|
| MM301 | warn | Same name in multiple scopes of one client — explain that client's precedence and which definition wins |
| MM302 | info | Same normalized identity (command+args vector, or URL) configured in ≥2 clients — inventory aid |
| MM303 | warn | Same identity across clients/scopes but **divergent** env key sets, args vectors, or headers key sets — likely drift (version-pin divergence is exclusively MM304) |
| MM304 | warn | Same npm package referenced with different pinned versions across entries |

### Security, static (MM4xx)

| ID | Sev | Logic |
|---|---|---|
| MM401 | error | Secret-looking literal in `env` values, `headers` values, or Cursor `auth.CLIENT_SECRET`: known prefixes (`sk-`, `ghp_`, `github_pat_`, `xoxb-`, `AKIA…`, `AIza…`, `ya29.`, JWT shape) or ≥32-char high-entropy token in a secret-named key (`*KEY*`, `*TOKEN*`, `*SECRET*`, `*PASSWORD*`). Evidence always masked (§12) |
| MM402 | error | POSIX: config file containing ≥1 detected secret is group/world-readable (mode & 0o077 ≠ 0). Without secrets: info variant. Windows: skipped |
| MM403 | error | Remote URL uses `http://` (or `ws://`) to a non-loopback host |
| MM404 | warn | Unpinned package execution: `npx`/`bunx` without `@<exact-version>` (`-y` alone or `@latest`), `uvx` without `==`/`@` pin — supply-chain drift |
| MM405 | info | Dynamic header/exec hooks present (`headersHelper`) — legitimate but arbitrary-shell-at-startup; surfaced for awareness |
| MM406 | warn | Command is a shell wrapper: `sh|bash|zsh -c`, `cmd /c`, `powershell -Command` — string-injection prone; recommend direct argv |
| MM407 | warn | Secret-looking literal inside `args` (visible in process listings) — recommend env |
| MM408 | error | Credentials embedded in URL (userinfo `user:pass@` or query param named like token/key with secret-looking value) |
| MM409 | warn | Gemini `trust: true` — bypasses tool-call confirmations for that server |
| MM410 | warn | `envFile` exists but is (POSIX) group/world-readable — a missing `envFile` is covered by MM205 |

### Hygiene (MM5xx)

| ID | Sev | Logic |
|---|---|---|
| MM501 | info | Entry explicitly disabled (`enabled=false`, disabled lists) — reminder it still holds possibly-stale secrets |
| MM502 | warn | Transport is deprecated SSE — migrate to streamable HTTP |
| MM503 | error | claude-code: server name in reserved list (`workspace`, `claude-in-chrome`, `computer-use`, `Claude Preview`, `Claude Browser`) — entry is skipped by the client |
| MM504 | info | Server name contains characters outside `[A-Za-z0-9_-]` — breaks Claude Code import and cross-client portability |
| MM505 | info | claude-code: `projects.<path>` entry defines `mcpServers` but `<path>` no longer exists — stale local-scope config |

### Probe, dynamic — only with `--connect` (MM6xx)

| ID | Sev | Logic |
|---|---|---|
| MM601 | error | Spawn failed (ENOENT/EACCES/EPERM) — includes errno, resolved command |
| MM602 | error | No successful `initialize` within `--timeout` ms; process killed (SIGTERM→SIGKILL); notes Codex default 10 s / client defaults |
| MM603 | error | Process alive but handshake failed: invalid initialize result, JSON-RPC error, version negotiation failure |
| MM604 | error | Process exited before/after handshake unexpectedly — exit code/signal + last 10 redacted stderr lines |
| MM605 | warn | Handshake OK but `tools/list` failed, or server advertises tools capability and returns zero tools |
| MM606 | error | HTTP probe: DNS failure, connection refused, TLS error, 5xx, 404 — classified detail |
| MM607 | info | HTTP probe: 401/403 without usable credentials — "requires interactive auth (OAuth); not verifiable non-interactively" |
| MM608 | error | stdio: non-JSON-RPC bytes on stdout before/among frames — protocol corruption (classic broken server) |
| MM609 | info | Startup succeeded but took > 5000 ms (configurable soft threshold) |
| MM610 | warn | Tool-name collision: same tool name exposed by ≥2 servers enabled in the same client — shadowing/confusion risk |
| MM611 | info | Server negotiated a protocol version older than the latest spec (2025-11-25) — informational |

### Online — only with `--online` (MM7xx)

| ID | Sev | Logic |
|---|---|---|
| MM701 | error | npx/npm package does not exist on registry.npmjs.org |
| MM702 | warn | Package exists but is deprecated (registry `deprecated` field) |
| MM703 | warn | Package name within Damerau-Levenshtein distance 1–2 of a curated list of popular MCP packages, and is not itself popular — possible typosquat |
| MM704 | info | Exact-pinned version ≥1 major behind `dist-tags.latest` |

---

## 11. Probe engine (dynamic checks)

### 11.1 Eligibility

Probe set = inventory entries where `enabled !== false`, transport ∈ {stdio, http, sse},
minus entries already failing MM101/MM102/MM105/MM106/MM201/MM204/MM206 (pointless to
spawn), minus extension-managed entries (Zed `source: extension`), minus
`--client`-filtered-out entries, de-duplicated by normalized identity
(§10 MM302) so the same server body is probed once even if configured in 4 clients
(result attached to every referencing entry). `ws` transport: not probed in v1
(recorded as `skipped`, visibility note).

### 11.2 stdio probe state machine

```
IDLE → SPAWNING → CONNECTING → INITIALIZED → LISTING → OK
   │        │           │            │           │
   │   spawn error  timeout/     handshake   tools/list
   │        ▼       exit          error        failure
   │   SPAWN_FAILED  ▼               ▼            ▼
   │              TIMEOUT      HANDSHAKE_ERR   LISTING_ERR (MM605; still OK-ish)
   └── always → SHUTDOWN: close transport → SIGTERM → 2 s grace → SIGKILL (process group)
```

Implementation requirements:
- `child_process.spawn(command, args, { shell: false })` — never `shell: true`.
- Env: `{ ...minimalBase, ...entry.env }` where `minimalBase` = `PATH`, `HOME`,
  `USERPROFILE`, `TMPDIR`/`TEMP`, `SystemRoot` (win32) — not full `process.env`
  (prevents leaking the operator's unrelated secrets into probed servers).
- Spawn in its own process group (`detached: true` on POSIX) so the kill escalation
  can target the group; on Windows use `taskkill /pid /T /F` fallback.
- stderr: 64 KiB ring buffer, redacted + sanitized before any output.
- Handshake via `@modelcontextprotocol/sdk` `Client` + `StdioClientTransport`;
  `initialize` params advertise client name `mcp-medic` and the SDK's protocol version.
- After `initialize`: `tools/list` (always), capture count + names (names feed MM610).
  `resources/list`/`prompts/list` are not called in v1 (cost/benefit).
- Per-probe wall-clock budget = `--timeout` (default 15 000 ms) covering
  spawn→listing; when Codex `startup_timeout_sec` is set for the entry, the effective
  budget is `min(--timeout, startup_timeout_sec × 1000)`.

### 11.3 HTTP/SSE probe

- `fetch` with `AbortSignal.timeout(timeout)`; try streamable HTTP `initialize` POST
  via SDK `StreamableHTTPClientTransport`; on 4xx/405 fall back to legacy SSE transport
  once (mirrors client behavior; result notes which succeeded).
- Static `headers` from the entry are sent as configured (they may embed tokens — that
  is the user's own config contacting the user's own endpoint). `headersHelper` is
  never executed (§2.4).
- Classification → MM606/MM607; TLS certificate errors are MM606 with `tls` class and
  remediation text (never an option to disable verification — no insecure mode exists).
- Redirect policy: follow same-origin redirects up to 3; cross-origin ⇒ MM606 detail.

### 11.4 Concurrency & isolation

- Pool of `--concurrency` (default 4, max 16) probes; per-probe timeout independent.
- Total probe phase budget: `timeout × ceil(n / concurrency) + 10 s` hard cap; on
  breach, remaining probes marked `skipped` with detail.
- Probes never run for `--format json --output` piping differences — behavior identical
  regardless of output format.

### 11.5 Online registry client

- Only endpoint: `https://registry.npmjs.org/<package>` (abbreviated metadata via
  `Accept: application/vnd.npm.install-v1+json`) — package name is the only data sent.
- 5 s timeout, 2 retries, in-memory cache per run, ≤4 concurrent requests.
- Package extraction: from `npx`/`bunx` args (first non-flag arg), parse
  `name[@version]` with scoped-name support.
- Failure to reach the registry ⇒ single MM-note (visibility note), not per-package
  errors, and never a non-zero exit by itself.

---

## 12. Redaction & output sanitization (`src/core/redact.ts`)

The one module through which all outward-bound strings pass.

**Secret detection** (shared with MM401/407/408): prefix patterns (see MM401), Shannon
entropy > 3.8 bits/char for candidates ≥ 32 chars in secret-named keys, JWT structural
match (`xxx.yyy.zzz` base64url).

**Masking**: `sk-ant-api03-…` → `sk-an…[REDACTED:41]` — keep ≤5 leading chars, append
`[REDACTED:<len>]`. Masking is applied to: all `env` values, all `headers` values,
`auth.*` values, any string flagged by detection anywhere (args, urls — url userinfo
and token-named query params are masked wholesale).

**Sanitization** (terminal-injection defense): strip/escape C0 control chars (except
`\n`, `\t`), C1, ANSI CSI/OSC sequences from every string that originated in config
files or process output before it reaches a renderer. JSON renderer keeps sanitized
values (control chars escaped by JSON anyway; sanitization still applied for defense
in depth when the JSON is later `cat`-ed).

**Guarantee (tested)**: for every fixture containing a known planted secret, no output
mode (text/json/markdown, any flag combination) contains the secret substring.

---

## 13. Reporting

- **text**: grouped by client → file → findings; severity-colored (`picocolors`),
  `NO_COLOR`/`--no-color` honored; ends with summary line + exit-code hint. Progress
  messages go to stderr, report to stdout (pipe-safe).
- **json**: `Report` (§4) serialized; schema exported to `schemas/report.schema.json`
  (generated from zod in build, committed, versioned by `schemaVersion`). Breaking
  schema changes require a major bump of `schemaVersion`.
- **markdown**: same content as text in GH-flavored tables (for pasting into issues/PRs).
- `list` renders the inventory table: client, scope, name, transport, target
  (command or host — path/args redact-checked), enabled.
- `clients` renders adapter table: id, detected?, files found, notes.
- `rules` renders the catalog from `rules/registry.ts` (single source, so docs cannot
  drift from code).

---

## 14. Security model

### 14.1 Trust boundaries

| Boundary | Direction | Treatment |
|---|---|---|
| Client config files | in | Untrusted input: tolerant parsing, size caps, no eval, all strings sanitized before output |
| Spawned server processes (`--connect`) | in/out | Untrusted code, executed **only** with explicit consent; hardened spawn (§11.2); output capped/redacted |
| Configured remote URLs (`--connect`) | out | Contacted only with consent; TLS verification never disabled; static headers sent as configured |
| npm registry (`--online`) | out | HTTPS only; sends package names only; responses schema-validated; no code execution |
| Terminal / report file | out | Redaction + sanitization guarantee (§12) |
| mcp-medic's own supply chain | — | Minimal deps, lockfile, pinned CI actions, npm provenance, no postinstall |

### 14.2 Abuse cases considered

1. **Tampered config as bait**: victim runs a doctor tool that auto-executes a
   malicious `command`. Defense: static-by-default; `--connect` prints the exact argv
   list before consent; disabled entries never probed. (This is the direct fix for the
   competitor's default-execute posture.)
2. **Terminal escape injection** via server name/stderr (`\e]0;…` window-title tricks,
   CSI overwrite): sanitization (§12).
3. **Secret exfiltration via shared reports**: users paste reports into issues.
   Defense: redaction on all paths; no "raw dump" mode exists in v1.
4. **Symlinked config pointing at sensitive files** (e.g. `mcp.json → /etc/shadow`):
   size caps, JSON/TOML parse failure ⇒ MM101 (content never echoed raw; parser
   messages only), profile-escape refusal for implicit candidates (§8).
5. **Malicious `--adapter` file**: zod-validated shape, paths expanded but only read,
   no code loading (declarative only).
6. **Registry response poisoning**: schema-validated fields only (`deprecated`,
   `dist-tags`, `versions` keys); strings sanitized before display.
7. **DoS**: 5 MiB file cap; probe wall-clock caps; bounded concurrency; ring-buffered
   process output.

### 14.3 Secure defaults summary

No exec, no network, no writes, no telemetry — each capability is a separate explicit
flag, and the only write target ever is `--output <file>` given by the user.

### 14.4 Release security

- npm publish with `--provenance` from GitHub Actions OIDC (no long-lived npm token in
  repo secrets if trusted publishing is available; else token scoped + 2FA).
- GitHub Actions pinned by commit SHA; minimal `permissions:` blocks; Dependabot for
  actions + npm; CodeQL workflow.
- `SECURITY.md`: private vulnerability reporting via GitHub advisories; 90-day
  disclosure target.
- No install scripts in the published package; `files` whitelist keeps the tarball to
  `dist/`, `schemas/`, `README`, `LICENSE`.

---

## 15. Error handling & failure modes

| Failure | Behavior |
|---|---|
| One adapter throws unexpectedly | Caught by orchestrator → visibility note + `--verbose` stack; other clients unaffected; exit code unaffected (defect, but degraded > dead) |
| Config unreadable (EACCES) | MM107 finding |
| Home dir undeterminable | exit 2 with explicit message |
| Probe child ignores SIGTERM | SIGKILL to process group after 2 s; if still alive (unkillable), report + continue |
| Registry unreachable | Single note; online rules skipped |
| Report `--output` unwritable | exit 2 |
| Ctrl-C during probes | Kill all children (group), print partial report notice, exit 130 |

---

## 16. Performance targets

- Static scan (8 adapters, ~50 servers): < 1 s wall on a warm disk.
- Memory: < 100 MiB RSS in static mode.
- Probe phase: bounded by §11.4 formula; default worst case with 20 stdio servers ≈
  20/4 × 15 s = 75 s (documented; `--timeout`/`--concurrency` tunable).

---

## 17. Testing & validation strategy

### 17.1 Layers

1. **Unit** (vitest): parsers, paths (per-platform via injected env/platform), redact
   (secret corpus + ANSI corpus), each rule with positive/negative cases, transport
   normalization, package-spec extraction, typosquat distance.
2. **Adapter contract tests**: one shared harness runs every adapter against its
   fixture set: `valid-full`, `valid-empty`, `missing-file`, `malformed`,
   `secrets-planted`, plus adapter-specific quirk fixtures (e.g. claude-code
   `url-no-type`, zed legacy command object, windsurf `serverUrl`).
3. **Probe integration**: scripted fake servers (`tests/fixtures/servers/*.mjs`):
   `ok.mjs` (normal SDK server, 2 tools), `slow.mjs` (sleeps > threshold, then ok),
   `hang.mjs` (never responds), `crash.mjs` (exits 1 after 100 ms),
   `garbage-stdout.mjs` (prints text before frames), `no-tools.mjs`,
   `stderr-noise.mjs` (secrets + ANSI in stderr — must come out redacted/sanitized).
   HTTP probe against an in-process `node:http` fixture (ok / 401 / 404 / 500 / TLS-bad
   via self-signed https server).
4. **E2E golden tests**: run built CLI against `tests/fixtures/homes/<case>` with
   `MCP_MEDIC_HOME`/`APPDATA`/`XDG_CONFIG_HOME` overridden; snapshot text and JSON
   outputs (timestamps normalized); assert exit codes per case.
5. **Security regression suite**: redaction guarantee (§12), spawn-hardening asserts
   (`shell:false`, minimal env), kill-escalation leaves no orphan (poll process table),
   consent gate (`--connect` non-TTY without `--yes` ⇒ exit 2).

Test-only seams: the CLI honors `MCP_MEDIC_FAKE_NOW` (fixed clock for deterministic
golden files) and the programmatic API accepts `registryBaseUrl` (mock registry).
Neither is a documented user flag; both are exercised only by the test suite.

### 17.2 CI matrix

GitHub Actions: {ubuntu-latest, macos-latest, windows-latest} × Node {20, 22, 24}:
lint, typecheck, unit+integration, build, E2E on built artifact. Windows runs the full
suite (path/spawn semantics differ — not best-effort).

### 17.3 Whole-product acceptance (v1 definition of done)

On a machine with fixture configs for all 8 clients containing 1 planted issue per rule
family, `mcp-medic scan --connect --online --yes --format json` must report every
planted issue (no false negatives on the fixture corpus), zero findings on the
`all-healthy` fixture home (no false positives), and never emit any planted secret.

---

## 18. Distribution & release

- npm package `mcp-medic`; `bin: {"mcp-medic": "dist/cli.js"}`; `type: module`;
  `engines: {"node": ">=20"}`; ESM only; tsup bundles CLI (deps inlined except SDK?
  — no: all runtime deps inlined for `npx` cold-start speed; SDK inlined too;
  package has zero runtime `dependencies` in the published tarball if bundling proves
  clean, else keep declared deps — decided at implementation, both acceptable).
- Versioning: SemVer from 0.1.0; `schemaVersion` decoupled (§13).
- Release flow: tag `vX.Y.Z` → CI builds, tests, publishes with provenance → GitHub
  Release with generated notes. npm account/token/trusted-publisher setup is a
  manual user step (documented in `docs/ISSUE_PLAN.md` handoffs).
- README: install/quickstart, consent-model explanation, rule table (generated from
  registry via script), CI usage snippet, security policy pointer.

---

## 19. v2 deferred (explicitly out of v1)

1. `mcp-medic fix` — interactive repair with backups (needs write-path design + ADR).
2. Client log analysis (Claude Desktop/Code `mcp*.log` crash evidence).
3. SARIF output for code-scanning integration.
4. Server migration/sync between clients ("copy this server to Cursor").
5. Additional clients: JetBrains, LM Studio, Cline/Roo, BoltAI, mcpm-managed setups.
6. Claude Desktop `.mcpb`/extension and claude.ai connector visibility (if APIs appear).
7. i18n (message catalog exists only if this lands).
8. `--only <rule>` selective runs, config file for defaults (`.mcpmedicrc`).
9. Watch/daemon mode.
10. Registry sources beyond npm (PyPI for `uvx` servers, Docker Hub tags).

## 20. Known unknowns (may spawn issues during implementation)

| # | Unknown | Planned resolution |
|---|---|---|
| U1 | Claude Code managed `managed-mcp.json` exact per-OS paths | Verify against docs/en/managed-mcp during adapter issue; keep scope `managed` behind a constant |
| U2 | VS Code non-default profiles & Insiders paths | v1 ships default-profile + Insiders candidates; profile enumeration deferred |
| U3 | Windsurf `serverUrl` vs `url` precedence when both present | Test against real Windsurf install; until then accept both, prefer `serverUrl`, note in adapter |
| U4 | Zed remote/context-server extension entries shape variations | Adapter tolerates unknown `source` values as inventory-only |
| U5 | Codex `.codex/config.toml` project-scope trust gating | Read file regardless; annotate "loaded only for trusted projects" in detail text |
| U6 | Windows process-group kill reliability | Integration-tested in CI windows runner; fallback `taskkill /T /F` |
| U7 | SDK behavior when server negotiates pre-2024-11-05 versions | Covered by MM603/MM611 classification tests against fake servers |
| U8 | Gemini `mcp-server-enablement.json` schema stability | Tolerant reader; absence ⇒ `enabled: 'unknown'` |
