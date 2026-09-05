# Title

Declarative custom adapters (`--adapter <file>`)

# Summary

Implement `src/discovery/custom.ts`: load a zod-validated JSON declaration that
describes an arbitrary client's config location and field mapping, and turn it into a
`ClientAdapter` at runtime — extending mcp-medic to unlisted clients without code.

# Context

DESIGN.md §7.4 fixes the declaration schema. This is the v1 answer for long-tail
clients (owner requirement: "その他" group). Declarative-only by design: no executable
plugin surface (security model §14.2 case 5).

# Scope

- `src/discovery/custom.ts` (schema + `loadCustomAdapter(filePath: string): ClientAdapter`)
- `tests/unit/discovery/custom.test.ts` + fixtures
  `tests/fixtures/custom-adapters/{valid.json,bad-schema.json,toml-variant.json}`

# Detailed Requirements

1. zod schema (exported as `customAdapterSchema`):
   ```
   id: string regex /^[a-z][a-z0-9-]{1,31}$/ and not one of the built-in ids
   displayName: string 1..64
   configFiles: array 1..8 of {
     path: string (supports ~ and ${ENV} via paths.expandTilde/expandEnvVars)
     scope: 'user' | 'project'
     format: 'json' | 'jsonc' | 'toml'
     serversPointer: string starting with '/' (JSON Pointer to an object of
       name→entry; for toml: dot-path string like 'mcp_servers' also accepted —
       validate: for toml allow /^[A-Za-z0-9_.-]+$/ instead)
     fieldMap: {
       command?: string; args?: string; env?: string;
       url?: string; headers?: string; type?: string; enabled?: string
     }  // values are property names inside each entry object
   }
   ```
2. `loadCustomAdapter(filePath): ClientAdapter` — declaration parsing/validation at
   load time; config-path resolution happens inside the generated
   `candidateFiles(ctx)`:
   - Read + parse the declaration (strict JSON), validate with zod. Any failure →
     throw `CustomAdapterError` with zod's flattened message (CLI maps to exit 2 —
     usage error, per DESIGN §2.3/§7.4).
   - Path resolution in `candidateFiles(ctx)` for every declared path, in order:
     `expandEnvVars` → `expandTilde` → if still not absolute and `scope: 'project'`,
     resolve against `ctx.projectDir`; a still-relative `user` path is a declaration
     error (throw `CustomAdapterError` at load time by static inspection where
     detectable, else surface at discovery as an adapter note).
   - Every generated `CandidateFile` carries `origin: 'custom'` (exempts these
     explicit paths from the discovery symlink-escape policy — DESIGN §7.1 /
     issue 08).
   - Generated `parseFile` honors the full §7.1 contract: malformed target file →
     `parseError` (no throw); `serversPointer` missing from the document → zero
     entries (healthy); pointer resolves to a non-object → `structuralIssues`
     `servers-key-not-object`; non-object entry → `entry-not-usable`; wrong-typed
     mapped fields → `field-wrong-type` with issue-09-style salvage defaults.
   - Field mapping: each key of the servers container = server name; apply
     `fieldMap` (missing mapped properties → field absent); transport via the
     generic `normalizeTransport` (client id is custom → the generic `unknown` row
     applies when ambiguous); `enabled` property read as boolean when mapped, else
     `'unknown'`.
   - Entries get `client: <declared id>`; pointer =
     `<serversPointer>/<RFC 6901-escaped name>` for json/jsonc. For TOML files the
     declaration uses a dot-path (`serversPointer: "mcp_servers"`), and
     `source.pointer` is still emitted as a JSON Pointer over the parsed value
     (`/mcp_servers/<escaped name>`), matching issue 12's convention.
3. Custom adapters receive generic rules only. Tests in this issue assert **recorded
   facts** (e.g. a url-without-type custom entry records no
   `clientSpecific.urlWithoutType` because that flag is claude-code-specific;
   transport is `unknown`); the corresponding rule firing/non-firing assertions
   (MM103 silent, MM106/MM401 active for custom clients) live in issues 20/23.
4. Registry integration: `--adapter` may be passed multiple times; duplicate ids
   (including collision with built-ins) → `CustomAdapterError`.

# Acceptance Criteria

- [ ] `valid.json` fixture (JSON client with `mcpServers`-like layout under a
      different key) produces correct entries through the standard contract harness
      (harness invoked with the generated adapter), including the malformed and
      missing-pointer cases from requirement 2.
- [ ] `toml-variant.json` maps a TOML file with dot-path declaration and emits
      JSON-Pointer `source.pointer` values that pass the harness resolution check.
- [ ] A server name containing `/` round-trips with correct RFC 6901 escaping.
- [ ] `bad-schema.json` (wrong scope enum) throws `CustomAdapterError` whose message
      names the offending path (`configFiles.0.scope`).
- [ ] Id collision with `codex` rejected.

# Validation

`npm test -- --run tests/unit/discovery/custom.test.ts`; reviewer checks schema
against DESIGN.md §7.4 field-for-field.

# Dependencies

08.

# Non-goals

- No executable/JS plugin adapters, no auto-detection of unknown clients, no
  client-specific rules for custom clients, no remote fetching of adapter files.

# Design References

- DESIGN.md §7.4, §14.2 case 5, §2.2 (`--adapter` flag)
