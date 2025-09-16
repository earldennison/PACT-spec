[← 04-queries](04-queries.md) | [↑ Spec Index](../README.md) | [→ 06-debugging](06-debugging.md)

# 05 – Snapshots and Diffs

## 1. Purpose
This section defines how **snapshots** are produced, addressed, and compared.  
Snapshots make context state reproducible, diff-able, and auditable.

---

## 2. Snapshot Definition
### 2.1 Commit
Each cycle produces exactly one snapshot.  
A snapshot is the PACT tree at the post‑commit boundary per Lifecycle §7.

Snapshots are one‑to‑one with turns: every turn is a committed snapshot of the entire context tree. A snapshot fixes the state of all segments in `^seq`, the system container (`^sys`), and the active head (`^ah`) at commit time. Snapshots are immutable.

### 2.1.1 Snapshot Boundary (Normative)
The snapshot boundary follows the Commit Sequence (Normative) in Lifecycle §7. For cycle `N`, the sealed snapshot for that cycle (addressable as `@t-1` after the next commit) MUST re‑serialize to exactly the provider input bytes for that cycle. Implementations MAY persist snapshots asynchronously, provided the logical content equals the post‑commit state used to produce those bytes.

Fresh‑@t0 equivalence: If no writes occur after a commit, the active working state `@t0` serializes identically to `@t-1`. Any write breaks the equivalence until the next commit.

### 2.2 Byte Stability
A snapshot MUST be byte-stable: given the same input, serialization yields identical bytes.  
Serialization MUST NOT mutate the snapshot.

### 2.3 Addressing
Temporal addresses:
- `@t0` — active working state (unsealed; not part of history)  
- `@t-1` — most recent sealed snapshot  
- `@cN` — absolute cycle number (sealed)

Range addressing across snapshots MUST be supported with inclusive semantics:

- `@t-<start>..<end>` — inclusive range using double-dots
- `@t-<start>:<end>` — inclusive range using a colon (interchangeable with `..`)

Notes:

- Negative indices are relative to the current snapshot: `@t-1` is the immediate previous snapshot, `@t-2` two cycles back, etc.
- `..` and `:` MUST be treated as interchangeable range operators and produce identical results.

### 2.4 Temporal Lens

- `@t0` = the active working state (unsealed; editable).  
- `@t-1`, `@t-2`, … = sealed snapshots.
- `@tA..@tB` (inclusive) = a temporal range. `@t0` MAY appear as an endpoint; in that case, the range includes the current working state evaluated at query time.  
- If no `@t` is given, selectors apply to the working state (the live active head).  
- Time‑derived facets and range semantics are valid whenever a temporal prefix is present; including `@t0` in a range is allowed. Determinism holds for a fixed input state; because `@t0` is mutable pre‑commit, repeated evaluations MAY differ if the working state changes between calls.

### 2.5 Read Model: Introspection (Normative)
For any node `N` in a snapshot (`@t0` or sealed):
- `up(N)` := `parent_id` (null for roots)
- `frame(N)` := `children(parent_id)` if `parent_id ≠ null`, else `∅`
- `index(N)` := resolved sibling position within `frame(N)` per canonical ordering (02 – Invariants §4.3)

---

## 3. Export and Import
### 3.1 Export
Implementations MUST support exporting snapshots in serialized form including:
- All nodes and required metadata,  
- Canonical sibling ordering (per [02 – Invariants §4.3](02-invariants.md)),  
- Stable IDs.
 
 

### 3.2 Import
Implementations SHOULD support re-importing snapshots for replay.  
Replay MUST yield the same serialization as the original.

---

## 4. Diffs
### 4.1 Identity
Diffs MUST use node `id` as the unit of identity.

### 4.2 Change Detection
A node is “changed” if any header field or its `content_hash` differs. TTL changes MUST be reported as modifications.

### 4.3 Output Format (Normative)
Diffs MUST return a JSON object with exactly these keys:
- `added` — array of node IDs newly present,  
- `removed` — array of node IDs no longer present,  
- `changed` — array of objects each of the form `{id: string, fields: string[]}` for modified nodes.

### 4.4 Order
Diff outputs MUST preserve canonical order of the newer snapshot (per [02 – Invariants §4.3](02-invariants.md)).

### 4.5 Rematerialization and ID Strategy
Intra-active-turn moves MAY appear as removed + added when implementations rematerialize nodes. Tools SHOULD optionally infer "moved" when content hashes match.

**Rationale**: Node IDs are stable within their lifetime, but implementations may rematerialize nodes during internal operations. Conformance is judged on snapshot consistency and rendered bytes, not on ID preservation across internal rebuilds.

---

## 5. Examples

### 5.1 Snapshot Export
```json
{
  "spec_version": "PACT/1.0.0",
  "cycle": 42,
  "root": {
    "id": "root-xyz",
    "children": [...]
  }
}
```

### 5.2 Diff Result (Normative)
```json
{
  "added": ["block:9a2f"],
  "removed": ["block:7c14"],
  "changed": [
    {"id": "block:5d8b", "fields": ["ttl", "priority"]}
  ]
}
```

### 5.3 Snapshot Range Addressing

Selectors MAY reference multiple temporal states using ranges. For example:

```
ctx.select("@t-5:@t0 ^ah .block")
```

This MUST include the working state at `@t0` and sealed snapshots `@t-1`…`@t-5` (inclusive), returning a RangeDiffLatestResult (see Selectors §3.7.1). If `@t0` changes between evaluations, results MAY differ accordingly; sealed endpoints remain stable.

## 6. Mutations (Aligned with Universal Depth)

This section defines mutation semantics against snapshots using Universal Depth (03 – Lifecycle §3.6). Mutations MUST preserve immutability of sealed bytes; history edits are expressed structurally.

### 6.1 insert(selector, at=depth=k)
- `k < 0` → insert into System space (depth<0). Requires authorization per implementation policy. This spec does not prescribe immutability for system nodes during the active cycle.
- `k = 0` → insert into the Active Head (writable this episode).
- `k > 0` → splice into history at depth `k` (shift `[k..∞] → [k+1..∞]`).
  - When inserting with `key=K`, if any live instances with `key=K` exist, enforce ODI such that every prior instance satisfies `depth(prev) < k`. Engines MUST auto‑demote prior instances or reject with `OverlapWouldViolateODI`.

### 6.2 replace(selector, with=component)
- For `depth > 0`, replacements MUST be non‑destructive: record an overlay or perform a splice that preserves prior bytes; sealed records are not mutated in place.
- For `depth = 0`, replacements mutate the current episode’s writable content.

### 6.3 move(selector, to_depth=k)
- `>0 → >0` ⇒ splice‑out from original depth and splice‑in at `k` (shifting as needed).
- `0 → >0` ⇒ early seal of moved nodes into history.
- `any → <0` ⇒ forbidden unless system‑capable (policy‑gated).
  - For overlapping instances sharing the same `key`, moves MUST preserve ODI; engines auto‑adjust or reject if violated.

### 6.4 delete(selector)
- `depth > 0` ⇒ record a redaction marker (non‑destructive history surgery).
- `depth = 0` ⇒ drop immediately (pre‑commit space).
- `depth < 0` ⇒ forbidden by default (policy‑gated).
  - Deletions do not affect ODI directly; normal TTL/commit rules apply.

Implementations SHOULD realize large-scale history surgery via logical indirection (e.g., an index) rather than physical renumbering of historical depths.

## 7. Conformance Checklist

An implementation is conformant if:

1. Each cycle MUST produce exactly one snapshot.
2. Snapshots MUST be byte-stable (identical input → identical bytes).
3. Snapshots MUST be addressable (@t0, @t-1, @cN).
4. Snapshot range addressing (`@tA..B` and `@tA:B`) MUST be supported with inclusive semantics and both operators MUST be interchangeable.
5. Export MUST include all nodes, required metadata, and canonical order.
6. Import MUST yield identical serialization output as original.
7. Diff identity MUST be based on node `id`.
8. Changes MUST be field-sensitive (detect header and content changes).
9. Diff outputs MUST preserve canonical order per newer snapshot.
10. Serialization MUST NOT mutate the snapshot (side-effect free).
11. Remove+add patterns for intra-active-turn changes are permitted for conformance.
12. Tools SHOULD detect logical moves via content hash matching where possible.

[← 04-queries](04-queries.md) | [↑ Spec Index](../README.md) | [→ 06-debugging](06-debugging.md)