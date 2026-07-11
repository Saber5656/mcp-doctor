# Title

Secret detection, masking, and output sanitization (`src/core/redact.ts`)

# Summary

Implement the single module through which all outward-bound strings pass: secret
pattern detection (shared by security rules), masking, and terminal-injection
sanitization — with a test corpus that later backs the product-wide "no secret ever
emitted" guarantee.

# Context

DESIGN.md §12 defines detection, masking, and sanitization. This module is a hard
dependency of the security rules (issue 23), all renderers (31–33), and the E2E
security regression suite (35). Client configs legitimately contain tokens; leaking
one in a report users paste into GitHub issues would be a product-defining failure.

# Scope

- `src/core/redact.ts`
- `tests/unit/core/redact.test.ts` + corpus files `tests/fixtures/redact/{secrets.txt,benign.txt,ansi.txt}`

# Detailed Requirements

1. `detectSecret(value: string, keyHint?: string): SecretHit | null` where
   `SecretHit = { reason: 'prefix' | 'jwt' | 'entropy'; pattern?: string }`:
   - Tokenization: candidates are maximal runs of `[A-Za-z0-9_\-.+/=]` within the
     value; each token is tested independently (so secrets embedded in URLs or
     `KEY=value` strings are found).
   - Prefix patterns, each a full regex over one token (case-sensitive):
     `/^sk-[A-Za-z0-9_-]{16,}$/`, `/^(ghp|gho|ghs|ghu|ghr)_[A-Za-z0-9]{20,}$/`,
     `/^github_pat_[A-Za-z0-9_]{20,}$/`, `/^xox[bpsar]-[A-Za-z0-9-]{10,}$/`,
     `/^xapp-[A-Za-z0-9-]{10,}$/`, `/^AKIA[0-9A-Z]{16}$/`, `/^ASIA[0-9A-Z]{16}$/`,
     `/^AIza[0-9A-Za-z_-]{35}$/`, `/^ya29\.[A-Za-z0-9_-]{20,}$/`,
     `/^glpat-[A-Za-z0-9_-]{20,}$/`, `/^npm_[A-Za-z0-9]{36}$/`,
     `/^pypi-AgEIcHlwaS5vcmc[A-Za-z0-9_-]*$/`.
   - JWT: token with exactly three dot-separated base64url segments whose first
     segment base64url-decodes to JSON parseable as an object containing an `alg`
     property (key order/whitespace irrelevant).
   - Entropy: token length ≥ 32, Shannon entropy > 3.8 bits/char, **and** `keyHint`
     matches `/key|token|secret|password|credential|auth/i` — all three required
     (prevents false positives on long paths/URLs).
2. `mask(value: string): string` → keep the first `Math.min(5, Math.floor(len / 4))`
   characters (0 for `len < 8`, i.e. short secrets are fully hidden) +
   `[REDACTED:<len>]`. Examples: `sk-ant-api03-…` (41 chars) → `sk-an[REDACTED:41]`;
   `abc123` → `[REDACTED:6]`.
3. `redactEntryForOutput(entry: ServerEntry): ServerEntry` — returns a deep copy
   where: every `env` value → masked unconditionally; every `headers` value → masked
   unconditionally; every `clientSpecific.secretCandidates[].value` → masked
   unconditionally; all other `clientSpecific` values traversed recursively
   (objects/arrays; string leaves tested with `detectSecret(leaf, <last path
   segment>)` and masked on hit); `args` elements masked when `detectSecret` hits;
   `url` → `maskUrl(url)`.
4. `maskUrl(url: string): string` — masks userinfo (`user:***@`) and the values of
   query params whose names match the secret-key regex; leaves everything else intact.
5. `sanitizeForTerminal(s: string): string` — removes/escapes: C0 controls except
   `\n`/`\t` (map to `␀`-style pictures or `\xNN` escapes — choose one, document),
   C1 range, ANSI CSI/OSC/DCS sequences (`\x1b[`, `\x1b]`, `\x1bP` … terminators),
   lone `\x1b`. Also hard-caps the string at 2 000 chars with `…[truncated]`.
6. Pure functions; no I/O; deterministic.

# Acceptance Criteria

- [ ] Every line of `secrets.txt` (≥ 25 realistic fakes covering all patterns above)
      is detected; `benign.txt` (≥ 25 lines: long paths, UUIDs, base64 images, git
      SHAs, URLs without creds) has zero detections.
- [ ] `mask` output never contains > 5 leading original chars; masked value never
      contains the full original.
- [ ] `maskUrl('https://u:p@h/x?api_key=abc123')` hides `p` and `abc123`, keeps host/path.
- [ ] `sanitizeForTerminal` corpus: OSC window-title attack, CSI cursor-move, raw
      `\x07` bell — none survive in output; `\n`/`\t` do.
- [ ] `redactEntryForOutput` never mutates its input (deep-copy asserted).

# Validation

`npm test -- --run tests/unit/core/redact.test.ts`; reviewer checks pattern list
against DESIGN.md §12 and MM401 table.

# Dependencies

01, 04 (`ServerEntry` type used by `redactEntryForOutput`; type-only import, but the
type must exist first).

# Non-goals

- No rule/Finding emission (issue 23 consumes detection), no renderer integration
  (32), no config-file scanning.

# Design References

- DESIGN.md §12 (redaction & sanitization), §10 MM401/MM407/MM408, §14.2 cases 2–3
