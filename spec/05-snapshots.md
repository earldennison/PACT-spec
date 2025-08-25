[← 04-selectors](04-selectors.md) | [↑ Spec Index](../README.md) | [→ 06-debugging](06-debugging.md)

# 05 – Snapshots and Diffs

## 1. Purpose
This section defines how **snapshots** are produced, addressed, and compared.  
Snapshots make context state reproducible, diff-able, and auditable.

---

## 2. Snapshot Definition
### 2.1 Commit
Each cycle produces exactly one snapshot.  
A snapshot is the PACT tree at the post‑commit boundary per Lifecycle §7.

### 2.1.1 Snapshot Boundary (Normative)
The snapshot boundary follows the Commit Sequence (Normative) in Lifecycle §7. For cycle `N`, the snapshot at `@t0` MUST re‑serialize to exactly the provider input bytes for that cycle. Implementations MAY persist snapshots asynchronously, provided the logical content equals the post‑commit state used to produce those bytes.

### 2.2 Byte Stability
A snapshot MUST be byte-stable: given the same input, serialization yields identical bytes.  
Serialization MUST NOT mutate the snapshot.

### 2.3 Addressing
Snapshots are referenced as:
- `@t0` — current snapshot  
- `@t-1` — previous snapshot  
- `@cN` — absolute cycle number

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
Intra-`^ah` moves MAY appear as removed + added when implementations rematerialize nodes. Tools SHOULD optionally infer "moved" when content hashes match.

**Rationale**: Node IDs are stable within their lifetime, but implementations may rematerialize nodes during internal operations. Conformance is judged on snapshot consistency and rendered bytes, not on ID preservation across internal rebuilds.

---

## 5. Examples

### 5.1 Snapshot Export
```json
{
  "spec_version": "PACT/0.1.0",
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
  "added": ["cb:9a2f"],
  "removed": ["cb:7c14"],
  "changed": [
    {"id": "cb:5d8b", "fields": ["ttl", "priority"]}
  ]
}
```

## 6. Conformance Checklist

An implementation is conformant if:

1. Each cycle MUST produce exactly one snapshot.
2. Snapshots MUST be byte-stable (identical input → identical bytes).
3. Snapshots MUST be addressable (@t0, @t-1, @cN).
4. Export MUST include all nodes, required metadata, and canonical order.
5. Import MUST yield identical serialization output as original.
6. Diff identity MUST be based on node `id`.
7. Changes MUST be field-sensitive (detect header and content changes).
8. Diff outputs MUST preserve canonical order per newer snapshot.
9. Serialization MUST NOT mutate the snapshot (side-effect free).
10. Remove+add patterns for intra-`^ah` changes are permitted for conformance.
11. Tools SHOULD detect logical moves via content hash matching where possible.

[← 04-selectors](04-selectors.md) | [↑ Spec Index](../README.md) | [→ 06-debugging](06-debugging.md)