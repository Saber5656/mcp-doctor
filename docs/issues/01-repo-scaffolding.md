# Title

Repository scaffolding: package, TypeScript, build, test, lint

# Summary

Create the complete TypeScript/Node project skeleton for `mcp-medic` — package
metadata, strict TS config, tsup build, vitest, ESLint/Prettier, MIT license — so that
every later issue starts from a working `npm run build && npm test` baseline.

# Context

The repository currently contains only `README.md`. DESIGN.md §3 fixes the module
layout, §3.2 the dependency policy, ADR-002 the stack, ADR-006 the license. The
npm package name is `mcp-medic` (ADR-001) even though the repo may still be named
`mcp-doctor` at implementation time.

# Scope

- `package.json`, `tsconfig.json`, `tsup.config.ts`, `vitest.config.ts`,
  `eslint.config.js`, `.prettierrc.json`, `.editorconfig`, `.gitignore`, `LICENSE`
- Exactly three source/test files (no other placeholders; later issues own the rest
  of the DESIGN §3.1 tree): `src/cli/main.ts`, `src/index.ts`,
  `tests/unit/smoke.test.ts`
- npm scripts: `build`, `dev`, `test`, `test:watch`, `test:coverage`, `lint`,
  `format`, `typecheck`

# Detailed Requirements

1. `package.json`:
   - `"name": "mcp-medic"`, `"version": "0.1.0"`, `"license": "MIT"`,
     `"type": "module"`, `"engines": {"node": ">=20"}`.
   - `"bin": {"mcp-medic": "dist/cli.js"}`; `"main"`/`"exports"`: `dist/index.js`
     (+ `./package.json` export).
   - `"files": ["dist", "schemas", "README.md", "LICENSE"]`.
   - No `postinstall`/`prepare` scripts that execute at consumer install time
     (`prepare` for local dev build is allowed only if it does not run on `npm i -g`
     of the published tarball — prefer `prepack`).
   - Runtime dependencies exactly: `@modelcontextprotocol/sdk@^1.29`, `commander@^15`,
     `zod@^4.4`, `smol-toml@^1.7`, `jsonc-parser@^3.3`, `picocolors@^1.1`.
   - Dev dependencies: `typescript@^5`, `tsup@^8.5`, `vitest@^4.1`,
     `@vitest/coverage-v8@^4.1` (backs the `test:coverage` script used by later
     issues' coverage criteria), `eslint@^9`, `typescript-eslint`, `prettier@^3`,
     `@types/node@^20`.
   - `"keywords"`: mcp, model-context-protocol, doctor, diagnostics, security, cli,
     claude, cursor, codex, vscode.
2. `tsconfig.json`: `"strict": true`, `"module": "NodeNext"`,
   `"moduleResolution": "NodeNext"`, `"target": "ES2022"`, `"noUncheckedIndexedAccess": true`,
   `"exactOptionalPropertyTypes": true`, `"verbatimModuleSyntax": true`, `outDir` unused
   (tsup builds), `"noEmit": true` for typecheck script.
3. `tsup.config.ts`: export `defineConfig([cliConfig, libConfig])` — two separate
   config objects, because the bundling policy differs per entry:
   - `cliConfig`: entry `src/cli/main.ts` → `dist/cli.js`; `format: ['esm']`;
     `banner: {js: '#!/usr/bin/env node'}`; `noExternal: [/.*/]` (bundle all runtime
     deps for npx cold-start speed); `dts: false`.
   - `libConfig`: entry `src/index.ts` → `dist/index.js`; `format: ['esm']`; deps
     external (tsup default); `dts: true`.
   - Both: `target: 'node20'`, `clean: true` only on `cliConfig` (or use a shared
     `outDir` with `clean` once — implementer's choice, documented in the config).
4. ESLint flat config with `typescript-eslint` recommended-type-checked; Prettier as
   formatter (no eslint-prettier conflicts); ignore `dist/`.
5. `LICENSE`: MIT, copyright line `Copyright (c) 2026 mcp-medic contributors`
   (matches ADR-006).
6. `.gitignore`: `node_modules/`, `dist/`, `coverage/`, `*.tsbuildinfo`, `.DS_Store`.
7. Placeholder `src/cli/main.ts`: reads name/version from its own package.json
   (embedded at build via tsup `define` or `createRequire` — choose one, document in
   code) and prints `mcp-medic <version>`; exits 0. Placeholder `src/index.ts`:
   `export async function scan(): Promise<never> { throw new Error('not implemented — see docs/issues/34-cli-wiring.md'); }`
   (issue 34 replaces it with the real signature).
8. One smoke test `tests/unit/smoke.test.ts` asserting the version string shape.
9. Commit `package-lock.json`.

# Acceptance Criteria

- [ ] `npm ci && npm run build` succeeds on Node 20 with no warnings from tsup.
- [ ] `node dist/cli.js` prints `mcp-medic 0.1.0` and exits 0.
- [ ] Local tarball run works: `npm pack`, then
      `npx --yes ./mcp-medic-0.1.0.tgz` prints the version line. The tarball
      contains only `dist/`, `README.md`, `LICENSE`, `package.json` (`schemas/` is
      listed in `files` in advance but the directory first appears in issue 31 —
      npm silently skips missing `files` entries).
- [ ] `npm test`, `npm run lint`, `npm run typecheck` all pass.
- [ ] No dependency outside the lists in Detailed Requirements (checked by reviewer).

# Validation

Run: `npm ci`, `npm run typecheck`, `npm run lint`, `npm test`, `npm run build`,
`node dist/cli.js`, `npm pack --dry-run` (inspect file list). All green on a clean
checkout.

# Dependencies

None (first issue).

# Non-goals

- No CI workflow (issue 02), no community files (issue 03), no real CLI commands
  (issue 34), no README rewrite (issue 36).

# Design References

- DESIGN.md §3 (architecture & layout), §3.2 (dependencies), §18 (distribution)
- ADR-001 (name), ADR-002 (stack), ADR-006 (license)
