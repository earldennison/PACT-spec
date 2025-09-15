# Changelog

## Unreleased
- Renamed canonical node types: `cb`→`block`, `mc`→`cont`, `mt`→`seg`.
- Canonical selector shape locked: `@time  (position)  { attributes }  [ behavior ]  @controls`.
- Added shorthands: `#key` (NEVER id), `+tag` (root implies deep search, postfix filters only).
- Added deep search operators: `?` single-step, `??` recursive, and `within(hops…) ?? { … }`.
- Added `@history` sugar for `@t0..@t-*`.
- Introduced controls: `@mode`, `@p`, `@limit`, `@order`, `@first`, `@unique`.
- Split attributes `{}` vs behavior `[]`; emphasized WS vs snapshot semantics.
- Python API now requires string selectors in examples (`ctx.select("…")`, `ctx.add("…","…")`).
- Breaking: Removed `role` attribute from spec, grammar, and examples. Migration: use `kind`, `+tag`, or `#key` to discriminate; update selectors and serialization accordingly.
- Selectors: allow depth range hop `dA..dB` (inclusive).
- Docs/Grammar: removed “sealed history” terminology; depth is structural-only; time via `@t` only.

## 0.1.0 – Initial public specification
- Canonical placement & ordering centralized in 02 §4
- Lifecycle and Snapshot Boundary defined (03 §2.4, 05 §2.1.1, 02 §8.5)
- Selectors language and grammar (04)
- Snapshots and diffs with byte-stable serialization (05)
- Budgeting and pruning clarified; removed Budget Metric requirement
- Debugging, adoption, and conformance checklists
