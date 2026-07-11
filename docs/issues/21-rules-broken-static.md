# Title

Broken-server static rules MM201–MM206 + package-spec utility

# Summary

Implement the six static broken-server rules (command resolution, executability,
relative paths, missing runtimes, dangling arg paths, malformed URLs) and the shared
`src/core/pkgspec.ts` utility that extracts npm/uv package specs from command lines
(reused by MM304, MM404, and the online rules).

# Context

DESIGN.md §10 (Broken table). These rules answer "would this even start?" without
executing anything. The pkgspec util is placed here because MM204's runner detection
and the version rules need one canonical parser for `npx -y pkg@1.2.3 …` shapes.

# Scope

- `src/core/pkgspec.ts`
- `src/rules/broken/mm201.ts` … `mm206.ts` (+ barrel)
- `tests/unit/core/pkgspec.test.ts`, `tests/unit/rules/broken/*.test.ts`

# Detailed Requirements

1. `pkgspec.ts` — `extractPackageSpec(command: string, args: string[]): PkgSpec | null`:
   ```ts
   interface PkgSpec { runner: 'npx'|'bunx'|'uvx'|'pipx'|null; name: string;
                       version: string | null;   // exact pin only ('1.2.3'); 'latest'/ranges → null + pinned:false
                       pinned: boolean; ecosystem: 'npm'|'pypi'; raw: string }
   ```
   - `npx`/`bunx` precedence: when a package option is present (`-p <x>`,
     `--package <x>`, `--package=<x>`), its **value** is the package spec (the first
     positional token is then the binary to run, not the spec); otherwise the first
     non-flag token is the spec (other flags: `-y`, `--yes`, `-q`, `--quiet`
     consume no value). Scoped names (`@scope/name@1.2.3`) parsed correctly (split
     on last `@` after index 0). Tests cover all three package-option forms.
   - `uvx`/`pipx run`: pypi specs `name==1.2.3` / bare name; `pipx run name`.
   - Direct `node`/`python` script paths → null.
   - Exhaustive table-driven tests (≥ 20 cases) including `npx --yes -p @a/b@2 cli-cmd`.
2. **MM201** (entry stdio, error): absolute `command` → `fs.exists` false; bare name →
   `fs.which` null. Detail distinguishes the two. Skips entries with
   `unresolvedVars` touching `command` (unexpanded `${VAR}` would false-positive —
   MM105 already covers) and entries with `clientSpecific.inventoryOnly` (all MM2xx
   rules skip inventory-only entries — issue 16). **Mutual exclusion with MM204**: when the missing command is
   a member of the known runtime/runner list (see MM204), MM204 fires instead of
   MM201 — the detection is shared, only the remediation differs.
3. **MM202** (entry stdio, error): exists but `isExecutable` false (POSIX). win32:
   rule inert (document).
4. **MM203** (entry stdio, warn): `command` neither absolute nor bare-name (contains a
   separator but not rooted), or any `args[i]`/`clientSpecific.envFile` is a relative
   path that looks path-like (`./`, `../`). Claude Desktop detail cites "must use
   absolute paths" from vendor docs.
5. **MM204** (entry stdio, error): the missing command (same `which`/`exists`
   detection as MM201) is a known runtime/runner:
   `npx`, `bunx`, `uvx`, `pipx`, `docker`, `node`, `python`, `python3`, `deno`, `bun`.
   Remediation is runtime-specific installation guidance ("`npx` missing means
   Node.js is not installed — install Node.js ≥ 20", "`uvx` ships with uv — install
   uv", etc. — a small remediation map in this module). Exactly one of MM201/MM204
   fires per missing command.
6. **MM205** (entry, warn): missing filesystem references: absolute-ish paths among
   `args` and `clientSpecific.envFile` that do not exist (heuristic: token starts
   with `/`, `~/`, or `[A-Za-z]:\\` and has no glob chars; `~` expanded via paths
   module); `clientSpecific.missingFileRefs` entries (windsurf — already verified
   missing by the adapter); and `clientSpecific.cwd` when
   `clientSpecific.missingCwd === true` (gemini — the adapter records the boolean,
   the path lives in `cwd`).
7. **MM206** (entry remote, error): `url` fails `new URL()` (detail = the thrown
   parse error) OR parses but scheme ∉ {http, https, ws, wss} (detail = the
   rejected scheme; `new URL` accepts `ftp:` etc., so scheme rejection is a
   separate branch with its own message).

# Acceptance Criteria

- [ ] pkgspec table tests all pass, including scoped+pinned, `--package=` form,
      `uvx pkg==0.5.0`, and non-package commands returning null.
- [ ] MM201/MM204 exclusivity matrix: missing `/opt/custom/server` → MM201 only;
      missing `npx` → MM204 only (with Node.js remediation); missing `uvx` → MM204
      only (uv remediation); present `npx` → neither. Exactly one finding per missing
      command (asserted count).
- [ ] MM202: POSIX — existing absolute command without the executable bit fires;
      executable command silent; win32 run emits no MM202.
- [ ] MM203 does not fire for bare names (`npx`, `node`).
- [ ] MM205 expands `~` and ignores flag-like tokens (`--config`).
- [ ] MM206 accepts `wss://`, rejects `ftp://` and `not a url`.

# Validation

`npm test -- --run tests/unit/rules/broken tests/unit/core/pkgspec.test.ts` on all
three OSes (PATHEXT/executability branches differ).

# Dependencies

19, 05.

# Non-goals

- No spawning (probe issues), no registry lookups (29), no version-comparison logic
  (issues 22/23/29 consume `PkgSpec`).

# Design References

- DESIGN.md §10 Broken (MM2xx); research doc §1 (absolute-path requirement)
