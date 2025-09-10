[← 08-reference-implementations](08-reference-implementations.md) | [↑ Spec Index](../README.md)

# 10 – Annex: Spec-wide Conformance Checklist

This annex aggregates all conformance requirements into one actionable checklist. A new runtime SHOULD be able to implement PACT without reading prose by following the items below end-to-end.

---

## A. Core Structure & Invariants (from 02 – Invariants)

MUST (mark one):
- [ ] Yes  [ ] No — Regions `^sys`, `^seq`, `^ah` exist under `^root`.
- [ ] Yes  [ ] No — Mutability is anchored to **Active Turn (`at`)**; `^ah` is structural only (no text implies inherent mutability for `^ah`).
- [ ] Yes  [ ] No — Nodes have required headers (`id`, `nodeType`, `offset`, `ttl`, `priority`, `cycle`, `created_at_ns`, `created_at_iso`, `creation_index`).
- [ ] Yes  [ ] No — Each `mt` has exactly one `mc @ offset=0`.
- [ ] Yes  [ ] No — Canonical sibling order enforced: `offset` → `created_at_ns` → `creation_index` → `id`.
- [ ] Yes  [ ] No — TTL expiry + cascade at commit; sealed cores immutable; snapshots immutable.
- [ ] Yes  [ ] No — If pruning is implemented, it is deterministic (**priority → age → id**).
- [ ] Yes  [ ] No — Serialization has no side effects; diffs compare by `id`; invalid placements rejected.

SHOULD (mark one):
- [ ] Yes  [ ] No — Cross‑region relocations realized as remove+add (no observable in‑place region mutation across snapshots).
- [ ] Yes  [ ] No — Reference safety preserved (or expiry deferred) when liveness is uncertain.

Reference: 02 – Invariants §§2–9, §11.

---

## B. Selectors (from 04 – Context Selectors)

MUST (mark one):
- [ ] Yes  [ ] No — Are roots `^sys`, `^seq`, `^ah`, `^root` supported?
- [ ] Yes  [ ] No — Are types `.mt`, `.mc`, `.cb` supported, and do `.cb:summary` and `[nodeType='cb:summary']` return identical results?
- [ ] Yes  [ ] No — Do ID selectors `#<id>` match exactly (case‑sensitive)?
- [ ] Yes  [ ] No — Are pseudos `:pre`, `:core`, `:post`, `:depth(n)`, `:first`, `:last`, `:nth(n)` implemented, including `:depth(n1,n2,...)` lists and `:depth(n1-n2)` ranges?
- [ ] Yes  [ ] No — Are attributes `[offset] [ttl] [priority] [cadence] [created_at_ns] [created_at_iso] [nodeType] [id] [role] [kind]` supported with `= != < <= > >=` and correct numeric/string semantics?
- [ ] Yes  [ ] No — Are combinators descendant (`A B`) and child (`A > B`) supported?
- [ ] Yes  [ ] No — Are `@t0`, `@t-1`, `@cN`, `@*` supported, with default `@t0` if omitted and no requirement for explicit `@t0`?
- [ ] Yes  [ ] No — Are snapshot ranges `@tA..B` and `@tA:B` supported with inclusive semantics, and do both operators (`..`, `:`) produce identical results?
- [ ] Yes  [ ] No — Are results ordered canonically and queries side‑effect free?
- [ ] Yes  [ ] No — Do invalid selectors raise errors?

Reference: 04 – Context Selectors §3–§7.

---

## C. Snapshots & Diffs (from 05 – Snapshots and Diffs)

MUST (mark one):
- [ ] Yes  [ ] No — Does each cycle produce exactly one snapshot?
- [ ] Yes  [ ] No — Does the snapshot boundary follow the Commit Sequence, with `@t0` re‑serializing to the exact provider bytes for that cycle (async persistence allowed)?
- [ ] Yes  [ ] No — Are snapshots byte‑stable and serialization side‑effect free?
- [ ] Yes  [ ] No — Does export include all nodes, required metadata, and canonical order?
- [ ] Yes  [ ] No — Do diffs use node `id` for identity?
- [ ] Yes  [ ] No — Do diff outputs use exactly: `added` (string[]), `removed` (string[]), `changed` ({id: string, fields: string[]}[]), preserving the newer snapshot’s order?

SHOULD (mark one):
- [ ] Yes  [ ] No — Does import/replay yield identical serialization as the original export?
- [ ] Yes  [ ] No — Are logical moves detected via content hash when remove+add patterns occur during the Active Turn?

Reference: 05 – Snapshots and Diffs §2–§6.

---

## D. Debugging & Inspection (from 07 – Debugging and Inspection)

MUST (mark one):
- [ ] Yes  [ ] No — Do nodes expose mandatory metadata fields (see A.2)?
- [ ] Yes  [ ] No — Are selectors pure and do they support all required features (see B)?
- [ ] Yes  [ ] No — Does `ctx.diff` report `added`/`removed`/`changed` with `{id, fields[]}` and preserve newer‑snapshot order?
- [ ] Yes  [ ] No — Is there no serialization path that mutates the tree? (see A.8)

SHOULD (mark one):
- [ ] Yes  [ ] No — Does snapshot export preserve canonical ordering and replay/import yield identical serialization?

Reference: 07 – Debugging and Inspection §2–§7.

---

## E. Commit Sequence (Normative Reference)

At each cycle's commit: 1. TTL expiry and cascading cleanup; 2. Pruning/compaction (if implemented) in canonical order; 3. Seal `^ah` into a new `mt` under `^seq`; 4. Produce snapshot `@t0`.

[← 08-reference-implementations](08-reference-implementations.md) | [↑ Spec Index](../README.md)



### Annex: Migration Flag (Non-normative)

Runtime configuration MAY include:
- select_range_returns_diff = true  (default)

If a deployment needs legacy behavior, setting false causes range selects to return a flat, ordered list of IDs as in prior versions; `ctx.rangeDiffLatest(selector, opts?)` SHOULD be provided as a separate API for diffs during the migration period.