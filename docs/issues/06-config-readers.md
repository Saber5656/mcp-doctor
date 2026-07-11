# Title

Tolerant JSONC/TOML readers with location tracking and read limits

# Summary

Implement `src/parsers/jsonc.ts`, `src/parsers/toml.ts`, and `src/parsers/limits.ts`:
safe file loading (size caps, permission capture) and tolerant parsing that returns
values plus per-node line/column locations and structured errors — never throws on bad
content.

# Context

DESIGN.md §8. Seven clients use JSON files — two of them (VS Code, Zed) are
JSONC-native by design — and one uses TOML (Codex). We parse **all** JSON sources
tolerantly as JSONC to avoid false "broken config" findings; separately, for clients
whose own parser is documented strict-JSON (exactly: `claude-desktop`, `claude-code`),
a file that parses only as JSONC is itself a defect (MM102 strict-JSON variant).
Location data feeds every `Finding.location`.

# Scope

- `src/parsers/limits.ts`, `src/parsers/jsonc.ts`, `src/parsers/toml.ts`
- `tests/unit/parsers/*.test.ts` + text fixtures under `tests/fixtures/parsers/`

# Detailed Requirements

1. `limits.ts` — exact exported types (issue 08 consumes them verbatim):
   ```ts
   export interface LoadedText { kind: 'ok'; text: string; sizeBytes: number;
                                 posixMode?: number }
   export type LoadError =
     | { kind: 'missing' }
     | { kind: 'too-large'; sizeBytes: number }
     | { kind: 'unreadable'; errno: string }        // EACCES / EPERM
     | { kind: 'io-error'; errno: string };         // EISDIR, ENOTDIR, EIO, …
   export function loadTextFile(path: string,
     opts?: { maxBytes?: number }): LoadedText | LoadError
   ```
   - `stat` first; `size > (opts.maxBytes ?? 5 * 1024 * 1024)` →
     `{kind:'too-large', sizeBytes}`.
   - Capture `posixMode` (`stat.mode & 0o777`) on POSIX; `undefined` on win32.
   - Read as UTF-8; strip BOM; do not rewrite the text otherwise — offsets must match
     the on-disk content with BOM removed.
   - `ENOENT` → `missing`; `EACCES`/`EPERM` → `unreadable`; every other fs error →
     `io-error` (never throws).
   - Mapping into `ConfigFileInfo` (performed by discovery, issue 08): `missing` →
     `exists: false`; the other three → `exists: true` + `loadError` field
     (DESIGN §4) consumed by MM107.
2. `jsonc.ts` — wrap `jsonc-parser`:
   - `parseJsonc(text): ParsedJsonc` where
     `ParsedJsonc = { value: unknown; errors: JsoncError[]; tree: Node | undefined; lineStarts: number[] }`
     using `parseTree` + `getNodeValue`; map `ParseError` codes to readable messages;
     `lineStarts` is the precomputed offset index of the input text.
   - `locate(parsed: ParsedJsonc, jsonPointer: string): {line, column} | undefined` —
     resolves the (RFC 6901-escaped) pointer against `parsed.tree` and converts the
     node offset to 1-based line/col via `parsed.lineStarts`.
   - `findDuplicateKeys(tree): Array<{pointer, key, line, column}>` — walk object
     nodes; report keys occurring more than once in the same object (MM108 input).
   - `isStrictJson(text): boolean` — `JSON.parse` try/catch; used to flag
     "parses only as JSONC" for strict-JSON clients (MM102 variant input).
3. `toml.ts` — wrap `smol-toml`:
   - `parseToml(text): { value: unknown } | { error: {message, line?, column?} }`;
     smol-toml throws `TomlError` with position info — map it, never rethrow.
   - `locateTomlKey(text, keyPath: string[]): {line, column} | undefined` — best-effort
     regex scan for the table header or key line (`[mcp_servers.name]` /
     `name = `). Supported: bare and simple dotted keys (`[mcp_servers.my-server]`).
     Unsupported (returns `undefined`, documented in JSDoc): quoted keys containing
     dots/spaces (`[mcp_servers."my.server"]`), inline tables, array-of-tables.
     Tests cover a dotted server name and a quoted-key fallback to `undefined`.
4. No module here may write files or log to stdout/stderr.

# Acceptance Criteria

- [ ] JSONC fixture with comments + trailing commas parses; same file reported
      `isStrictJson === false`.
- [ ] Broken JSON fixture returns error with correct 1-based line/column
      (assert exact numbers).
- [ ] Duplicate-key fixture (`{"a":1,"a":2}` nested at depth ≥2) is detected with
      pointer `/x/a` style output.
- [ ] TOML fixture with duplicate table `[mcp_servers.x]` twice returns a structured
      error (not a throw) with line info.
- [ ] 6 MiB fixture (generated in test setup, not committed) returns `too-large`.
- [ ] EACCES simulated via `chmod 000` (POSIX-only test, skipped on win32) returns
      `unreadable`.

# Validation

`npm test -- --run tests/unit/parsers` green on macOS/Linux/Windows CI (issue 02
matrix); fixture files reviewed against DESIGN.md §8.

# Dependencies

01.

# Non-goals

- No schema validation of config content (adapters/rules), no YAML, no writes,
  no symlink policy decisions (discovery-level, issue 08).

# Design References

- DESIGN.md §8 (parsing layer), §10 MM101/MM102/MM107/MM108
