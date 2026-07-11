# ADR-007: CLI output is English-only in v1 (no i18n layer)

- Status: accepted (2026-07-11, approved by owner)

## Context

~40 diagnostic rules each carry title/detail/remediation strings. An i18n catalog
doubles authoring and review cost per rule and adds an abstraction layer to every
output path. The project is OSS-first; English is the ecosystem lingua franca.

## Decision

1. All CLI output, rule messages, JSON report strings, and error messages are English.
2. No message-catalog indirection in v1 — rule strings live with rule definitions.
3. Repository README may be bilingual (English primary, Japanese section allowed).
4. i18n is a v2 item; if it lands, it introduces a catalog then (accepting the
   refactor cost consciously rather than pre-paying it).

## Consequences

- Rule authoring stays single-file and reviewable.
- Japanese-speaking users read English diagnostics (mitigated by precise file/line
  references and copy-pasteable remediations).
