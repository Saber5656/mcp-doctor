# Research: MCP Client Configuration Formats

Status: verified 2026-07-11 against official documentation and local installations.
Scope: the eight clients targeted by mcp-medic v1, plus protocol-level facts that drive the probe engine design.

This document is the factual foundation for the client adapters in `docs/DESIGN.md`.
Every adapter issue in `docs/issues/` must conform to the facts recorded here.
If implementation reveals a discrepancy, update this file first, then the design.

## Protocol facts

- Current MCP specification version: **2025-11-25** (versions are date-strings; prior revisions: 2024-11-05, 2025-03-26, 2025-06-18). Version negotiation happens in the `initialize` handshake; client and server must agree on one version.
- Transports: **stdio** and **Streamable HTTP** are current. **SSE is deprecated** as a standalone transport. Claude Code additionally supports a proprietary `ws` (WebSocket) type.
- Official TypeScript SDK: `@modelcontextprotocol/sdk` **1.29.0** (as of 2026-07-11), engines `node >= 18`. Provides `Client`, `StdioClientTransport`, `StreamableHTTPClientTransport`, and legacy `SSEClientTransport` — sufficient for the probe engine without reimplementing the protocol.

Sources:
- https://modelcontextprotocol.io/specification/versioning
- https://modelcontextprotocol.io/quickstart/user

## Summary matrix

| Client | Config format | User/global scope path (macOS) | Project scope path | Top-level key | Transport field style |
|---|---|---|---|---|---|
| Claude Desktop | JSON | `~/Library/Application Support/Claude/claude_desktop_config.json` | — | `mcpServers` | stdio only in file (`command`/`args`/`env`) |
| Claude Code | JSON | `~/.claude.json` (`mcpServers` = user scope; `projects.<abs path>.mcpServers` = local scope) | `.mcp.json` at project root | `mcpServers` | `type`: `stdio` \| `http` \| `sse` \| `ws` (+ alias `streamable-http` for `http`) |
| Codex CLI | TOML | `~/.codex/config.toml` | `.codex/config.toml` (trusted projects only) | `[mcp_servers.<name>]` tables | stdio (`command`) or streamable HTTP (`url`) |
| Cursor | JSON | `~/.cursor/mcp.json` | `.cursor/mcp.json` | `mcpServers` | `type` for stdio; remote via `url` |
| Windsurf | JSON | `~/.codeium/windsurf/mcp_config.json` | — | `mcpServers` | stdio (`command`) or remote (`serverUrl`) |
| VS Code | JSON with comments | user-profile `mcp.json` (macOS: `~/Library/Application Support/Code/User/mcp.json`) | `.vscode/mcp.json` | `servers` | `type`: `stdio` \| `http` |
| Zed | JSON with comments | `~/.config/zed/settings.json` | `.zed/settings.json` | `context_servers` | stdio (`command`/`args`/`env`, `source`) |
| Gemini CLI | JSON | `~/.gemini/settings.json` | `.gemini/settings.json` | `mcpServers` | stdio (`command`), SSE (`url`), streamable HTTP (`httpUrl`) |

Windows equivalents are listed per client below. All paths must be resolved through a
single path-resolution module that honors `HOME`, `USERPROFILE`, `APPDATA`, and
`XDG_CONFIG_HOME` overrides so tests can run against a fake home directory.

## Per-client details

### 1. Claude Desktop (`claude-desktop`)

- Paths:
  - macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
  - Windows: `%APPDATA%\Claude\claude_desktop_config.json`
  - Linux: not officially supported; best-effort `~/.config/Claude/claude_desktop_config.json`.
- Shape: `{ "mcpServers": { "<name>": { "command": string, "args": string[], "env": {..} } } }`.
- The file may exist without an `mcpServers` key (observed locally: top-level keys `coworkUserFilesPath`, `preferences` only). Absence of the key is not an error.
- Remote servers ("connectors") and Desktop Extensions are managed in the claude.ai UI / extension store, **not** in this file. mcp-medic can only see file-based servers; this must be stated in output ("connectors not visible to this tool").
- MCP logs: macOS `~/Library/Logs/Claude/mcp.log` and `mcp-server-<NAME>.log`; Windows `%APPDATA%\Claude\logs\`. (v2 candidate input; out of v1 scope.)
- Known failure modes documented by Anthropic: relative paths not allowed (must be absolute), `${APPDATA}` unexpanded on Windows (fix: add `APPDATA` to `env`), `npx` failing when npm not installed globally.

Source: https://modelcontextprotocol.io/quickstart/user

### 2. Claude Code (`claude-code`)

The most complex adapter: five definition sources with precedence.

- Scopes and storage:
  | Scope | Stored in | Notes |
  |---|---|---|
  | local (default) | `~/.claude.json` under `projects.<absolute project path>.mcpServers` | private, per-project |
  | project | `.mcp.json` at project root | checked into VCS, needs user approval |
  | user | `~/.claude.json` top-level `mcpServers` | all projects |
  | plugin | plugin `.mcp.json` / `plugin.json` | out of v1 scope (inventory note only) |
  | managed | `managed-mcp.json` — macOS `/Library/Application Support/ClaudeCode/`, Linux `/etc/claude-code/`, Windows `C:\Program Files\ClaudeCode\` (per code.claude.com/docs/en/managed-mcp; re-verify at implementation) | managed `managed-settings.json` lives in the same directories; `allowedMcpServers`/`deniedMcpServers` policy applies to managed servers |
- Precedence when the same name appears in several scopes: local > project > user > plugin > claude.ai connector. Entire entry wins; no field merging.
- Server entry fields: `type` (`stdio` | `http` | `sse` | `ws`; `streamable-http` accepted as alias for `http`), `command`, `args`, `env`, `url`, `headers`, `headersHelper` (shell command that emits headers JSON — arbitrary code execution surface), `timeout` (per-tool-call ms, values < 1000 ignored), `alwaysLoad`, `oauth` (`clientId`, `callbackPort`, `authServerMetadataUrl`, `scopes`).
- **A `url` entry without `type` is a configuration error**: Claude Code reads it as stdio and skips the server (`MCP server "<name>" has a "url" but no "type"`). This is a high-value diagnostic rule.
- Env expansion in `.mcp.json` and `~/.claude.json`: `${VAR}` and `${VAR:-default}` in `command`, `args`, `env`, `url`, `headers`. **If a referenced variable is unset and has no default, Claude Code fails to parse the config** — another high-value rule.
- SSE transport is deprecated (warning-level rule).
- Reserved server names (config entry is skipped with a warning): `workspace`, `claude-in-chrome`, `computer-use`, `Claude Preview`, `Claude Browser`.
- Enable/disable state for `.mcp.json` servers lives in settings files: `enabledMcpjsonServers`, `disabledMcpjsonServers`, `enableAllProjectMcpServers` in `.claude/settings.json`, `.claude/settings.local.json`, `~/.claude/settings.json`, managed settings. `disabledMcpjsonServers` rejects regardless of source. Workspace-trust gates approvals from repo-tracked settings files.
- Startup timeout: `MCP_TIMEOUT` env var (ms) at client level.
- claude.ai connectors are fetched from the cloud and never appear in local files (same visibility caveat as Claude Desktop).
- CLI: `claude mcp list|get|add|add-json|add-from-claude-desktop|remove|reset-project-choices|login|logout|serve`.
- Local evidence (2026-07-11): `~/.claude.json` exists with 123 `projects` entries and no top-level `mcpServers`; project entries may lack `mcpServers` entirely. Adapter must treat missing keys as empty, not as errors. File permissions observed `0600`.

Source: https://code.claude.com/docs/en/mcp (fetched 2026-07-11)

### 3. Codex CLI (`codex`)

- Paths: `~/.codex/config.toml` (global); `.codex/config.toml` project scope (loaded for trusted projects only).
- Shape (TOML):
  ```toml
  [mcp_servers.docs]           # server name = table key
  command = "npx"
  args = ["-y", "docs-server"]
  env = { API_KEY = "..." }    # stdio only
  startup_timeout_sec = 10     # optional, default 10
  tool_timeout_sec = 45        # optional
  enabled = true               # optional; false disables without deleting

  [mcp_servers.remote]
  url = "https://example.com/mcp"          # streamable HTTP
  bearer_token_env_var = "MCP_TOKEN"       # optional; token read from env
  ```
- Local ground truth: installed `codex-cli 0.141.0`; `codex mcp add` exposes `--env`, `--url`, `--bearer-token-env-var`, `--oauth-client-id`, `--oauth-resource`. CLI: `codex mcp list|get|add|remove|login|logout`.
- TOML parse errors (e.g. duplicate table names) make the whole config invalid — Codex refuses to start sessions; syntax rule severity: error.
- Note: `bearer_token_env_var` is the secure pattern (no literal secret in file). A literal bearer token in `env` or headers is the anti-pattern to flag.

Sources: https://developers.openai.com/codex/mcp , https://developers.openai.com/codex/config-reference , local `codex mcp --help` (0.141.0)

### 4. Cursor (`cursor`)

- Paths: `~/.cursor/mcp.json` (global), `.cursor/mcp.json` (project).
- Top-level key: `mcpServers`.
- stdio fields: `type: "stdio"` (documented as required), `command` (required), `args`, `env`, `envFile` (stdio only).
- Remote fields: `url` (required), `headers`, `auth` object with `CLIENT_ID` (required), `CLIENT_SECRET`, `scopes` (static OAuth credentials — a secret-in-file surface).
- Transports: stdio, SSE, Streamable HTTP.
- Enable/disable: UI toggle (state not in `mcp.json`; adapter reports `enabled: unknown`).

Source: https://cursor.com/docs/context/mcp (fetched 2026-07-11)

### 5. Windsurf (`windsurf`)

- Path: `~/.codeium/windsurf/mcp_config.json` (single central file; no project scope documented).
- Top-level key: `mcpServers`.
- stdio fields: `command`, `args`, `env`. Remote fields: **`serverUrl`** (Windsurf-specific key; docs also show `url` in places — adapter must accept both, prefer `serverUrl`), `headers`.
- Transports: stdio, SSE, Streamable HTTP.
- Variable interpolation: `${env:VAR_NAME}` (environment) and `${file:/path/to/file}` (file contents) — different syntax from Claude Code; expansion rules are per-client.
- Enable/disable: UI-managed (adapter reports `enabled: unknown`).
- Org note: Windsurf docs now live under Cognition (`docs.windsurf.com` redirects to `docs.devin.ai`).

Source: https://docs.windsurf.com/windsurf/cascade/mcp (redirects to docs.devin.ai; fetched 2026-07-11)

### 6. VS Code (`vscode`)

- Paths:
  - Workspace: `.vscode/mcp.json`.
  - User profile: `mcp.json` in the user profile folder (macOS default profile: `~/Library/Application Support/Code/User/mcp.json`; Windows: `%APPDATA%\Code\User\mcp.json`; Linux: `~/.config/Code/User/mcp.json`). Non-default profiles store it under `User/profiles/<id>/mcp.json` — v1 scans the default profile and documents the limitation.
  - VS Code Insiders uses `Code - Insiders` in place of `Code` (v1: include as additional candidate paths).
- Top-level key: **`servers`** (not `mcpServers`). Optional sibling key `inputs` (secret prompts; values are stored by VS Code elsewhere, referenced as `${input:id}`).
- Per-server fields: `type` (`stdio` | `http`), `command`, `args`, `env`, `envFile`, `url`, `headers`, `sandboxEnabled` (boolean).
- Settings keys (in `settings.json`, not `mcp.json`): `chat.mcp.discovery.enabled` (imports servers from other apps — can cause "duplicate" observations), `chat.mcp.autoStart`.
- Enable/disable state stored outside `mcp.json` (adapter reports `enabled: unknown`).
- Files are JSONC (comments and trailing commas tolerated) — parser must be JSONC-tolerant.

Source: https://code.visualstudio.com/docs/copilot/customization/mcp-servers (fetched 2026-07-11)

### 7. Zed (`zed`)

- Paths: `~/.config/zed/settings.json` on macOS and Linux (Zed uses `~/.config` on macOS too); Windows: `%APPDATA%\Zed\settings.json`. Project scope: `.zed/settings.json`.
- Top-level key: `context_servers`.
- Per-server fields (custom servers): `source: "custom"`, `command` (string), `args`, `env`. Extension-provided servers use `source: "extension"` with `settings` object. Older documented shape nested the command as an object (`"command": { "path": ..., "args": [...] }`) — adapter must accept both shapes.
- `settings.json` is JSONC (comments allowed).
- Enable/disable: UI-managed per server.

Sources: https://zed.dev/docs/ai/mcp (URL verified via search results 2026-07-11; page content cross-checked against multiple integration guides)

### 8. Gemini CLI (`gemini-cli`)

- Paths: `~/.gemini/settings.json` (user), `.gemini/settings.json` (project); system-level settings exist for enterprise (out of v1 scope).
- Top-level key: `mcpServers`, plus a global `mcp` object (`serverCommand`, `allowed`, `excluded`).
- Per-server fields: exactly one of `command` (stdio) | `url` (SSE) | `httpUrl` (streamable HTTP) required; optional `args`, `env` (supports `$VAR_NAME` / `${VAR}` expansion), `cwd`, `headers`, `timeout` (ms, default 600000), `trust` (boolean, default false — `true` bypasses tool-call confirmations: security-relevant), `includeTools`, `excludeTools`.
- Enablement state: `gemini mcp enable|disable <name>` persists into `~/.gemini/mcp-server-enablement.json` (adapter should read this to report `enabled`).

Source: https://raw.githubusercontent.com/google-gemini/gemini-cli/main/docs/tools/mcp-server.md (fetched 2026-07-11)

## Cross-client observations that shape the design

1. **Three different top-level keys** (`mcpServers`, `servers`, `context_servers`) and one TOML dialect → a normalized `ServerEntry` model is mandatory; adapters own all format quirks.
2. **Env/e-var expansion syntax differs per client** (`${VAR}` / `${VAR:-def}` vs `${env:VAR}` / `${file:...}` vs `$VAR`); expansion must be adapter-scoped, never global.
3. **JSONC tolerance required** for VS Code and Zed (and harmless for the rest); a strict `JSON.parse` would produce false "broken config" findings.
4. **Remote-server URL key differs**: `url` (most), `serverUrl` (Windsurf), `httpUrl` vs `url` (Gemini distinguishes transport by key name).
5. **Enabled/disabled state is out-of-file for 4 of 8 clients**; the model needs `enabled: true | false | 'unknown'`.
6. **Secrets legitimately appear in these files** (env values, headers, `auth.CLIENT_SECRET`, bearer tokens) → the redaction layer is not optional; it gates every output path.
7. **Arbitrary-code-execution fields exist beyond `command`**: Claude Code `headersHelper`, Gemini `trust: true`, VS Code `chat.mcp.discovery` interplay — these deserve dedicated awareness rules.
8. **Client-side visibility limits**: claude.ai connectors (Claude Desktop/Code) and UI-managed extension stores are invisible to a file scanner; the report must say so explicitly to avoid a false sense of completeness.
9. **Duplicate detection across clients** must compare normalized identity: (a) same name, (b) same `command + args` vector, (c) same URL — after per-client expansion.
10. **Local evidence caveat**: files can legitimately exist with zero MCP entries; absence of keys is healthy, absence of files is healthy — only malformed content or dangling references are findings.
