# ADR-003: Static by default; dynamic probing only behind `--connect` (+ preview + confirmation)

- Status: accepted (2026-07-11, approved by owner)

## Context

"Broken server" detection is strongest when the tool actually spawns configured
commands and performs the MCP handshake. But a diagnostic tool that executes commands
found in config files **by default** is a security anti-pattern: the config under
diagnosis may be tampered with, pasted from an untrusted source, or contain exactly the
malicious entry the user wants investigated. The existing competitor (`mcp-doctor` on
npm) runs servers by default with an opt-out flag; we consider that a defect, not a
feature to copy.

Options considered:
- A) **Static by default; `--connect` enables probing — chosen.**
- B) Dynamic by default with a TTY confirmation — "Enter-key fatigue" still executes
  malicious commands; non-TTY behavior becomes ambiguous.
- C) Always dynamic, no confirmation — rejected outright.

## Decision

1. `mcp-medic scan` performs no process execution and no network I/O.
2. `--connect` enables probing, with a printed execution plan (exact argv / URL per
   server) and one interactive confirmation; non-TTY requires `--yes` (else exit 2).
3. Disabled entries are never probed. `headersHelper`-style config-defined commands are
   never executed even under `--connect`.
4. Probe hardening is mandatory: `shell:false` spawn, minimal base env, process-group
   kill escalation, output caps, per-probe timeout.

## Consequences

- First-run UX costs one flag; README/quickstart teach `npx mcp-medic --connect` as the
  "full checkup" invocation.
- CI usage is deliberate: `--connect --yes`.
- This is a headline differentiator and is documented in README and SECURITY.md.
