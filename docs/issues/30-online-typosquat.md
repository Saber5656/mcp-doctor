# Title

Typosquat heuristic + curated package list (MM703)

# Summary

Implement `src/online/typosquat.ts`: a curated list of well-known MCP-related npm
package names plus a Damerau-Levenshtein similarity check, emitting MM703 when a
configured package is suspiciously close to — but not — a known package.

# Context

DESIGN.md §10 MM703. Typosquatting (`@modelcontextprotocol/server-filesytem`) is a
realistic MCP-ecosystem attack: users hand-copy `npx` specs from READMEs. The rule is
`--online`-gated because verdicts should consider registry existence (a typo that
does not exist is already MM701; MM703 adds "…and it is 1 edit away from X" — and
flags the scarier case where the typo **does** exist as a real package).

# Scope

- `src/online/typosquat.ts` (`damerauLevenshtein`, `KNOWN_PACKAGES`, `findSquatCandidate`)
- `src/rules/online/mm703.ts`
- `tests/unit/online/typosquat.test.ts`

# Detailed Requirements

1. `damerauLevenshtein(a, b): number` — full DL (insert/delete/substitute/transpose),
   O(len a × len b); table-driven Vitest cases only (no property-testing dependency —
   issue 01's dependency policy).
2. `KNOWN_PACKAGES: Array<{name: string; verifiedAt: string}>` — curated, committed.
   Initial list (implementer verifies each name exists via
   `npm view <name> name` at implementation time, stamps `verifiedAt`, drops any
   that vanished, and records the verification output in the PR):
   `@modelcontextprotocol/sdk`, `@modelcontextprotocol/inspector`,
   `@modelcontextprotocol/server-filesystem`, `@modelcontextprotocol/server-memory`,
   `@modelcontextprotocol/server-everything`,
   `@modelcontextprotocol/server-sequential-thinking`,
   `@modelcontextprotocol/server-brave-search`, `@modelcontextprotocol/server-github`,
   `@modelcontextprotocol/server-postgres`, `@modelcontextprotocol/server-puppeteer`,
   `@modelcontextprotocol/server-slack`, `@modelcontextprotocol/server-google-maps`,
   `@playwright/mcp`, `firecrawl-mcp`, `@upstash/context7-mcp`,
   `@browserbasehq/mcp`, `@notionhq/notion-mcp-server`, `@sentry/mcp-server`,
   `@supabase/mcp-server-supabase`, `@elastic/mcp-server-elasticsearch`,
   `airtable-mcp-server`, `mcp-remote`, `@bytebase/dbhub`, `graphlit-mcp-server`.
   Additions welcome during implementation; every entry must be
   registry-verified.
3. `findSquatCandidate(name): {known: string; distance: 1 | 2} | null`:
   - Compare case-insensitively; scoped names compared both full and on the
     name-after-slash part (`server-filesytem` vs `server-filesystem`).
   - Distance 1 always a candidate; distance 2 only when the **compared candidate
     string** (after normalization — i.e. whichever of full-name or after-slash part
     is being compared) is ≥ 10 chars.
   - Exact member of `KNOWN_PACKAGES` → null (never flag the real thing).
4. **MM703** (warn): for entries with npm PkgSpec and `--online` data present:
   `findSquatCandidate` hit AND the configured name is not itself a well-known
   package AND the package's `OnlineData.npm` status is not `'unavailable'`
   (no-data → silent, consistent with all MM7xx rules). Detail states both names,
   the distance, and the registry existence of the configured name (exists → "a
   real package 1 edit from a popular one — verify you meant it"; not-found →
   cross-reference MM701). Remediation: the corrected spec string.
5. False-positive guard: names that are prefixes/suffixes of known names with
   distance ≥ 3 stay silent; add a denylist-of-flagging for known legitimate forks
   (empty initially, structured for additions).

# Acceptance Criteria

- [ ] DL tests: (`server-filesytem`, `server-filesystem`) = 1 (single deletion),
      (`mcp-medic`, `mcp-medik`) = 1 (substitution), (`abc`, `acb`) = 1
      (transposition), (`abc`, `cba`) = 2; symmetric; zero for equal strings.
- [ ] `@modelcontextprotocol/server-filesytem` flags against
      `…/server-filesystem` (scoped comparison path).
- [ ] `@playwright/mcp` (exact known) never flags.
- [ ] Short-name guard: `mcp` (len 3) never flags at distance 2.
- [ ] MM703 firing test wired through OnlineData for exists/not-found variants;
      `'unavailable'` status → silent (explicit test).

# Validation

`npm test -- --run tests/unit/online/typosquat.test.ts`; reviewer spot-checks 5 random
`KNOWN_PACKAGES` entries against the live registry.

# Dependencies

29.

# Non-goals

- No popularity/download-count queries, no homoglyph/unicode confusable analysis (v2
  candidate), no automatic list updates.

# Design References

- DESIGN.md §10 MM703; ADR-004; prior-art doc (differentiation table)
