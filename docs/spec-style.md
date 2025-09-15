# Spec Style Guide

## Principles
- Use RFC 2119 keywords: MUST, SHOULD, MAY.
- Prefer declarative, deterministic language.
- Keep normative rules centralized and referenced (e.g., placement/order in 02 §4).

## Structure
- Use numbered sections: `## 1.`, `### 1.1`.
- Include navigation links at top and bottom of each file.
- Provide short conformance checklists per chapter.

## Formatting
- Use backticks for code-like terms (`^sys`, `seg`, `cont`, `block`, `offset`).
- Use fenced code blocks for examples.
- Keep examples minimal and self-contained.

## Selector Formatting (SPA order)
- Canonical order: `@time  (position)  { attributes }  [ behavior ]  @controls`.
- Normalize attribute order: `key, id, type, kind, tag, schema, mime, lang`.
- Normalize behavior order: `ttl, cad, phase, age, born_turn`.
- Single-hop MAY be parenless only if previously agreed; otherwise use parentheses for multi-hop.
- Attach hop-scoped `{…}` with no preceding space; global `{…}`/`[ … ]` preceded by a single space.

## Versioning
- Reference current spec version from `VERSION` file.
- Include `spec_version` in snapshot exports where relevant.


