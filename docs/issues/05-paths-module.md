# Title

Cross-platform path resolution module (`src/core/paths.ts`)

# Summary

Implement the single module that resolves every filesystem location used by discovery —
home, APPDATA, XDG config — with environment-variable overrides that make the whole
tool testable against fake home directories.

# Context

DESIGN.md §6 requires that adapters never call `os.homedir()` directly and that setting
`MCP_MEDIC_HOME` / `APPDATA` / `XDG_CONFIG_HOME` redirects all discovery. This is the
foundation for the E2E fixture strategy (§17.1 layer 4).

# Scope

- `src/core/paths.ts`
- `tests/unit/core/paths.test.ts`

# Detailed Requirements

1. Export a factory `createPaths(env: NodeJS.ProcessEnv, platform: NodeJS.Platform)`
   returning an object with:
   - `home(): string` — `env.MCP_MEDIC_HOME` → `env.HOME` → `env.USERPROFILE` →
     throw `PathResolutionError('cannot determine home directory')`.
   - `appData(): string` — win32: `env.APPDATA` → `join(home(), 'AppData', 'Roaming')`;
     darwin: `join(home(), 'Library', 'Application Support')`;
     linux/other: `xdgConfig()`.
   - `xdgConfig(): string` — `env.XDG_CONFIG_HOME` (absolute only, else ignored per
     XDG spec) → `join(home(), '.config')`.
   - `expandTilde(p: string): string` — replaces leading `~/` or bare `~` with
     `home()`; all other inputs returned unchanged.
   - `expandEnvVars(p: string): string` — expands `${NAME}` from `env` (used only for
     custom-adapter paths, issue 18); unknown names left literal.
2. Absolute-path guarantee applies to `home()`, `appData()`, and `xdgConfig()` only
   (`expandTilde`/`expandEnvVars` are string transforms and may return relative
   strings unchanged). Env-derived roots are validated: a **relative** value in
   `MCP_MEDIC_HOME`, `HOME`, `USERPROFILE`, or `APPDATA` is ignored and resolution
   falls through to the next source (same policy the XDG spec mandates for
   `XDG_CONFIG_HOME`) — covered by tests.
   Use `path.win32`/`path.posix` selection driven by the injected `platform` so tests
   can exercise win32 behavior on any host.
3. No caching across calls (env may be mutated by tests); no filesystem access in this
   module — existence checks AND symlink/`realpath` display resolution (DESIGN §6)
   belong to the discovery layer (issue 08).
4. Export `PathResolutionError extends Error` with `name = 'PathResolutionError'`
   and `code: 'EPATHS'`; prototype fixed so `instanceof` works across ESM builds
   (`Object.setPrototypeOf(this, new.target.prototype)`); tests assert
   `instanceof`, `.name`, `.code`, and message content.

# Acceptance Criteria

- [ ] With `platform='win32'`, `env={APPDATA: 'C:\\Users\\u\\AppData\\Roaming', USERPROFILE: 'C:\\Users\\u'}`:
      `appData()` returns the APPDATA value; removing APPDATA falls back to the join.
- [ ] With `platform='darwin'`, `env={HOME:'/Users/u'}`: `appData()` =
      `/Users/u/Library/Application Support`.
- [ ] `MCP_MEDIC_HOME` beats `HOME` on every platform.
- [ ] Relative `XDG_CONFIG_HOME` is ignored (falls back to `~/.config`).
- [ ] Missing all home vars throws `PathResolutionError` (CLI maps it to exit 2 later).
- [ ] Relative values in home/APPDATA env vars are ignored (fall-through tested per
      variable).
- [ ] `npm run test:coverage -- tests/unit/core/paths.test.ts` reports 100% branch
      coverage for `src/core/paths.ts` (coverage tooling from issue 01).

# Validation

`npm test -- --run tests/unit/core/paths.test.ts` with a matrix of
(platform × env permutations) table-driven tests; typecheck green.

# Dependencies

01.

# Non-goals

- No per-client paths (each adapter composes its own from these helpers), no file
  existence/stat logic, no symlink/realpath handling (discovery, issue 08), no
  Windows registry lookups.

# Design References

- DESIGN.md §6 (path resolution), §17.1 (testing layers)
