[← 05-snapshots](05-snapshots.md) | [↑ Spec Index](../README.md) | [→ 07-adoption](07-adoption.md)

# 06 – Debugging and Inspection

## 1. Purpose
This section defines how PACT implementations MUST support **observability**:  
- what metadata nodes carry for inspection,  
- how selectors and diffs are exposed for debugging,  
- and how determinism guarantees reproducible audits.

---

## 2. Metadata Requirements
### 2.1 Mandatory Fields
Every node MUST carry at least:
- `id` — unique, stable identifier  
- `nodeType` — canonical or user-assigned (see Glossary: NodeType)  
- `offset` — relative offset within parent  
- `ttl` — remaining or expiry cycle  
- `priority` — pruning order  
- `cycle` — cycle in which node was created  
- `created_at_ns` — monotonic timestamp  
- `created_at_iso` — ISO8601 string  

### 2.2 Optional Fields
Nodes MAY also carry:
- `role` (e.g., `"user"|"assistant"|"tool"|"system"`)  
- `kind` (e.g., `"text"|"call"|"result"`)  
- `content_hash` (stable hash of payload for diffing)  
- `provenance` (who/what produced the node, e.g., `"model:gpt-5"`)  

### 2.3 Immutability
Metadata attached at creation MUST remain stable across cycles, except for `ttl` when decremented or evaluated.

### 2.4 Attribute Stability Table (Normative)

| Field            | Type     | MUST/MAY | Stability (Immutability)                 | Ordering Effect |
|------------------|----------|----------|------------------------------------------|-----------------|
| `id`             | string   | MUST     | Immutable within a snapshot               | Used for identity; not part of sibling order keys beyond tie-break in §4.3 |
| `nodeType`       | string   | MUST     | Immutable                                 | None |
| `offset`         | number   | MUST     | Immutable post-creation within a cycle    | Primary sibling order key (§4.3) |
| `ttl`            | number∣null | MUST  | MAY change by lifecycle evaluation        | None |
| `priority`       | number   | MUST     | Immutable unless policy updates           | Pruning policy only; not ordering |
| `cycle`          | number   | MUST     | Immutable                                 | None |
| `created_at_ns`  | number   | MUST     | Immutable                                 | Secondary sibling order key (§4.3) |
| `created_at_iso` | string   | MUST     | Immutable                                 | None |
| `creation_index` | number   | MUST     | Immutable                                 | Tertiary sibling order key (§4.3) |
| `role`           | string   | MAY      | Immutable                                 | None |
| `kind`           | string   | MAY      | Immutable                                 | None |
| `content_hash`   | string   | MAY      | MAY change when content changes           | None (explicitly excluded) |
| Namespaced custom (`data_*`, `content_*`) | any | MAY | SHOULD remain stable per meaning | None; unknown attributes MUST NOT affect canonical ordering |

---

## 3. Selector Inspection
### 3.1 Purity
`ctx.select` MUST be pure:  
Given a fixed snapshot and selector, the result MUST always be the same list of nodes in canonical order (per [02 – Invariants §4.3](02-invariants.md)).

### 3.2 Supported Features
Selectors MUST support:
- Roots: `^sys`, `^seq`, `^ah`, `^root`  
- Types: `.mt`, `.mc`, `.cb`  
- IDs: `#<id>`  
- Pseudos: `:pre`, `:core`, `:post`, `:depth(n)`, `:first`, `:last`, `:nth(n)`  
- Attributes: `[offset] [ttl] [priority] [cycle] [created_at_ns] [created_at_iso] [nodeType] [id] [role] [kind]`  
- Combinators: descendant (`A B`), child (`A > B`)  

### 3.3 Debug Queries
Examples:
```text
# all pre-context blocks in active head
ctx.select("^ah :pre .cb")

# last two historical turns, assistant responses only
ctx.select("^seq .mt:depth(1,2) .cb[role='assistant']")

# expired or soon-to-expire nodes
ctx.select("^root .cb[ttl<=1]")
```

---

## 4. Diff Inspection
### 4.1 Identity by ID

ctx.diff(A,B,selector) MUST compute added/removed/changed nodes using id as identity.

### 4.2 Change Detection

A node is “changed” if any header field or its content_hash differs between snapshots. TTL changes MUST be reported as modifications.

### 4.3 Diff Output (Normative Reference)

Diff outputs MUST conform to the schema defined in 05 – Snapshots and Diffs §4.3 Output Format (Normative). Implementations MUST NOT deviate from that JSON shape.

### 4.4 Debug Integration

Diff outputs MUST preserve canonical order of the newer snapshot (per [02 – Invariants §4.3](02-invariants.md)).
Debuggers MUST be able to map diffs back to serialized provider threads

---

### 5. Snapshot Replay
### 5.1 Export

Implementations MUST support exporting snapshots in a serialized form that includes:

All nodes with required metadata,

Canonical ordering preserved (per [02 – Invariants §4.3](02-invariants.md)),

Stable IDs.

### 5.2 Import

Implementations SHOULD support re-importing a snapshot for replay.
Replay MUST yield the same serialization output as the original.

---

## 6. Auditing Guarantees

Snapshots are byte-stable: given the same input, serialization is identical.

Selectors are pure: repeated queries yield the same results.

Diffs are complete: all changes are attributable to TTL, pruning, or mutation during the Active Turn.

Debugging is structural: the same invariants apply regardless of LLM stochasticity.

---

## 7. Conformance Checklist

Nodes expose all mandatory metadata fields.

Selectors support all required features and are pure.

Diffs use IDs as identity, report added/removed/changed.

Snapshots can be exported with preserved ordering.

Replay of snapshots yields identical serialization.

No serialization path mutates the tree.

Debugging and diffs are deterministic across runs.

### Errors specific to Snapshot Ranges

- E_SNAPSHOT_RANGE_KIND_MISMATCH — mixed "@t..@c" endpoints are invalid.
- E_SNAPSHOT_RANGE_WILDCARD — "@*" is not allowed inside a range.
- E_SNAPSHOT_RANGE_LIMIT — expanded snapshots exceed maxSnapshots.
- E_SELECTOR_INVALID — malformed selector or grammar violation.

[← 05-snapshots](05-snapshots.md) | [↑ Spec Index](../README.md) | [→ 07-adoption](07-adoption.md)
