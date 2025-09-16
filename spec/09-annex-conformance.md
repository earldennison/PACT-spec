[← 08-reference-implementations](08-reference-implementations.md) | [↑ Spec Index](../README.md)

# 10 – Annex: Spec-wide Conformance Checklist

This annex aggregates all conformance requirements into one actionable checklist. A new runtime SHOULD be able to implement PACT without reading prose by following the items below end-to-end.

Clarification: PACT governs structural invariants, addressing, selection, and snapshot/commit behavior only. Policies and implementations for node deletion, pruning, retention, or archival are intentionally out of scope for PACT and are left to implementations and application-layer policies.

---

## A. Core Structure & Invariants (from 02 – Invariants)

MUST (mark one):
- [ ] Yes  [ ] No — Regions `^sys`, `^seq`, `^ah` exist under `^root`.
- [ ] Yes  [ ] No — Mutability is anchored to **Active Turn (`at`)**; `^ah` is structural only (no text implies inherent mutability for `^ah`).
- [ ] Yes  [ ] No — Nodes have required headers (`id`, `nodeType`, `offset`, `ttl`, `priority`, `cycle`, `created_at_ns`, `created_at_iso`, `creation_index`).
- [ ] Yes  [ ] No — Each `seg` has exactly one `cont @ offset=0`.
- [ ] Yes  [ ] No — Canonical sibling order enforced: `offset` → `created_at_ns` → `creation_index` → `id`.
- [ ] Yes  [ ] No — Core container bytes are immutable after commit; snapshots are immutable.
- [ ] Yes  [ ] No — `parent_id` is tracked (or derivable) and enforced as the reference frame for `offset`; move/re‑parent semantics follow 02 – Invariants §4.5.1.
- [ ] Yes  [ ] No — Every node MUST include a `parent_id` attribute except for the global root. `parent_id` is the canonical frame of reference for offsets and ordering; offsets are relative to the referenced parent. Implementations MAY synthesize `parent_id` for imported content, but serialized nodes in snapshots MUST preserve explicit `parent_id` values for deterministic ordering and rehydration.
- [ ] Yes  [ ] No — Serialization has no side effects; diffs compare by `id`; Invalid selectors MUST raise errors.

SHOULD (mark one):
- [ ] Yes  [ ] No — Cross‑region relocations realized as remove+add (no observable in‑place region mutation across snapshots).

Reference: 02 – Invariants §§2–9, §11.

Removed tests: PACT conformance does not include tests for node pruning, expiry, or retention. Any tests asserting pruning or expiry behavior are out-of-scope for the core PACT conformance suite and should be moved to vendor-specific or implementation-specific test suites.

Mandatory conformance tests (core) — updated:

- Canonical ordering unit test — sibling sort-order deterministic.
- Byte-stable serialization test — serializing the same snapshot twice must produce identical bytes.
- Depth / range fixtures — selection by depth/range returns deterministic IDs given a snapshot.
- Selector alias test — selector aliases are equivalent (e.g., depth(0) ↔ ^ah).
- Invalid placement rejection — creating an invalid placement (e.g., duplicate container offset) returns the designated error code.

---

## B. Queries (from 04 – Context Queries)

MUST (mark one):
- [ ] Yes  [ ] No — Are roots `^sys`, `^seq`, `^ah`, `^root` accepted as structural inputs (normalized output prefers `depth(n)`/`.seg`)?
- [ ] Yes  [ ] No — Are types `.seg`, `.cont`, `.block` supported, and do `.TypeName` anchors normalize to `[nodeType='TypeName']`?
- [ ] Yes  [ ] No — Do ID selectors `#<id>` match exactly (case‑sensitive)?
- [ ] Yes  [ ] No — Are predicates `:pre`, `:core`, `:post`, `:first`, `:last`, `:nth(n)` implemented?
- [ ] Yes  [ ] No — Are attributes `[offset] [ttl] [cad] [priority] [cycle] [created_at_ns] [created_at_iso] [nodeType] [id]` supported with `= != < <= > >=` and correct numeric/string semantics?
- [ ] Yes  [ ] No — Are structural hops for descendant (`A B`) and child (`A > B`) supported?
- [ ] Yes  [ ] No — Are `@t0`, `@t-1`, `@cN`, `@*` supported, with default working state if omitted?
- [ ] Yes  [ ] No — Are snapshot ranges `@tA..B` and `@tA:B` supported with inclusive semantics, and do both operators (`..`, `:`) produce identical results?
- [ ] Yes  [ ] No — Are results ordered canonically and queries side‑effect free?
- [ ] Yes  [ ] No — Do invalid selectors raise errors?

Reference: 04 – Context Queries §3–§7.

---

## C. Snapshots & Diffs (from 05 – Snapshots and Diffs)

MUST (mark one):
- [ ] Yes  [ ] No — Does each cycle produce exactly one snapshot?
- [ ] Yes  [ ] No — Does the snapshot boundary follow the Commit Sequence, with the sealed snapshot (`@t-1`) re‑serializing to the exact provider bytes for that cycle (async persistence allowed)?
- [ ] Yes  [ ] No — Are snapshots byte‑stable and serialization side‑effect free?
- [ ] Yes  [ ] No — Does export include all nodes, required metadata, and canonical order?
- [ ] Yes  [ ] No — Do diffs use node `id` for identity?
- [ ] Yes  [ ] No — Do diff outputs use exactly: `added` (string[]), `removed` (string[]), `changed` ({id: string, fields: string[]}[]), preserving the newer snapshot’s order?
- [ ] Yes  [ ] No — On structural moves, diffs include `parent_changed {previous_parent_id, new_parent_id}` and `offset_changed {previous_offset, new_offset}` (at minimum).

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

At each cycle's commit: 1. TTL expiry and cascading cleanup; 2. Pruning/compaction (if implemented) in canonical order; 3. Seal the context into a snapshot; 4. Expose `@t0` as the new active working set.

---

## F. Region Alias Equivalence

To claim compliance with Region Alias Equivalence, an implementation MUST:

1) Accept `depth(0)` wherever `^ah` is accepted and produce identical selections.  
2) Accept structural depth forms for anchors (`depth(0)`) as identical to `^ah`. History uses `@t…`.
3) `^sys` is a structural anchor outside the depth sequence.  
4) Provide a conformance test that round-trips alias ↔ depth forms with identical results.

---

## Annex – Additional Compliance Items

### Lifecycle — MUST
1) Active Turn permits arbitrary edits (add/remove/re-parent/replace) within invariants.  
2) Commit produces a new `seg` at `depth(1)`; prior `seg` at `depth(k)` shift to `k+1`.  
3) `cont[offset=0]` bytes are immutable after commit.

### Selectors — MUST
1) Depth addressing via `dN`/`dA..dB` hops and structural anchors are supported.  
2) SnapshotRange yields `RangeDiffLatestResult` by default.  
3) No alternate “legacy” result shapes are required or recognized pre‑v1.

---

### Example JSON node

[See non-normative annex: Adopter extensions — `kind` historical examples]


## Annex – Editorial Process & Conformance Notes

### Single-Source Normative References
- Normative definitions for the depth axis, alias equivalence, and commit/immutability reside in:
  - 01 – Architecture / Regions & Addressing
  - 03 – Lifecycle / Active Turn & Commit
  - 04 – Context Queries / Overview + Snapshot & Range Semantics
- Other chapters MUST link to these as “Normative Reference: see …” rather than re‑stating. Remove duplicate normative paragraphs and replace with a one‑line normative reference.

Action
- DELETE any duplicated lifecycle checklists and selector fixtures that restate the above norms; replace each with:
  “Normative Reference: see the canonical section listed in ‘Single‑Source Normative References.’”

### Pre‑v1 Policy (Normative Reminder)
- No Migration Plan: The spec defines only the current normative behavior. Historical/legacy modes are NOT part of conformance pre‑v1.
- Change Control: Editors MAY introduce breaking changes that supersede prior drafts. Conformance is always evaluated against the latest draft.

[← 08-reference-implementations](08-reference-implementations.md) | [↑ Spec Index](../README.md)



### Annex: Migration Flag (Non-normative)

Runtime configuration MAY include:
- select_range_returns_diff = true  (default)

 
