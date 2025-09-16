[← 06-debugging](06-debugging.md) | [↑ Spec Index](../README.md) | [→ 08-reference-implementations](08-reference-implementations.md)

# 07 – Adoption and Compatibility

### Annex — Budgeting Guidance (Non-Normative)
Budgeting decides **when** to trigger pruning based on limits such as token count,
node count, depth, or application heuristics. PACT does **not** standardize budgeting.
When budgeting triggers pruning, conformant implementations MUST:
- Apply TTL expiry first, then pruning.
- Use canonical pruning order: **priority → age (`created_at_ns` asc) → id (lexicographic)**.
- Preserve determinism: given identical inputs and policies, produce identical post-state and bytes.

Examples of budgeting knobs (illustrative only): `max_tokens_per_region`, `max_nodes_per_turn`,
`max_depth(^seq)`, `priority_bands`.

## 1. Purpose
This section provides guidance on **adopting PACT**, ensuring compatibility across implementations,  
and extending the model without breaking determinism or interoperability.

---

## 2. Incremental Adoption
### 2.1 Minimal Runtime
A minimal compliant PACT runtime MUST implement:
- Root regions (`^sys`, `^seq`, `^ah`) under `^root`.  
- Canonical node types (`seg`, `cont`, `block`).  
- Deterministic sibling ordering.  
- TTL expiry and cascading cleanup.  
- Snapshot commit and serialization.  
- Basic selector support (`ctx.select`) and diffs (`ctx.diff`).

### 2.2 Incremental Layers
Implementations MAY adopt features in layers:
1. **Core structure** (tree + regions + turns).  
2. **Lifecycle** (TTL, expiry, sealing).  
3. **Optional pruning/compaction** (non-normative budgeting; pruning is optional for conformance).  
4. **Selectors & diffs** (debugging, inspection).  

Each layer adds determinism and observability without invalidating earlier layers.

---

## 3. Compatibility Rules
### 3.1 Canonical Types
Implementations MUST always expose canonical types (`seg`, `cont`, `block`),  
even if internal class names differ (e.g., `TextContextComponent` → `.block`).

### 3.2 User-Assigned Types
Implementations MAY define user-assigned types (e.g., `block:summary`).  
These MUST remain queryable via canonical selectors (`.block`) and attributes.

### 3.3 Opaque IDs
Node `id` values are opaque. UUIDs, hashes, or other schemes MAY be used.  
All implementations MUST treat IDs as case-sensitive opaque strings for compatibility.

### 3.4 Extensions
Extensions (new attributes or node types) MUST NOT break invariants.  
They MUST use namespacing (`block:foo`, `custom:bar`) to avoid collisions.  
Selectors MUST be able to target them via attributes.

---

## 4. Migration from Logs
### 4.1 Flat Logs
Flat conversation logs MAY be imported into PACT by mapping:
- `system` → `^sys .block +system`  
- `user` → `^ah .cont > .block +user` (then sealed to `^seq`)  
- `assistant` → `^ah .cont > .block +assistant` (then sealed)  

Minimal example (import → seal):

```json
{
  "flat_log": [
    {"tag": "system", "content": "You are helpful."},
    {"tag": "user", "content": "Hello"},
    {"tag": "assistant", "content": "Hi!"}
  ]
}
```

Imported into PACT prior to commit:

```json
{
  "root": {"children": [
    {"id": "sys-1", "nodeType": "^sys", "children": [
      {"id": "block:sysA", "nodeType": "block", "kind": "text", "offset": 0, "content": "You are helpful."}
    ]},
    {"id": "ah-1", "nodeType": "^ah", "children": [
      {"id": "cont:1", "nodeType": "cont", "offset": 0, "children": [
        {"id": "block:u1", "nodeType": "block", "kind": "text", "content": "Hello"},
        {"id": "block:a1", "nodeType": "block", "kind": "text", "content": "Hi!"}
      ]}
    ]}
  ]}
}
```

At commit, `^ah` is sealed into `^seq` as a new `seg`.

### 4.2 RAG and External Stores
External retrieval-augmented memory MAY insert items as `block` nodes.  
These nodes MUST respect TTL, pruning, and canonical ordering.

Example (RAG insert with TTL):

```json
{
  "id": "block:rag1",
  "nodeType": "block",
  "kind": "text",
  "content": "Doc: How to reset a password...",
  "ttl": 2,
  "offset": 1
}
```

Placement guidance:
- Attach RAG `block` under the relevant turn’s post‑context (`offset > 0`) or into `^ah` during the current cycle.
- Use `ttl` to expire retrievals deterministically; pruning SHALL follow priority → age → id.

---

## 5. Versioning
### 5.1 Spec Version
Snapshots SHOULD include a `spec_version` field (e.g., `"PACT/1.0.0"`).  
This ensures replay across different runtimes is compatible.

### 5.2 Forward Compatibility
- Older implementations MUST ignore unknown attributes or node types.  
- Extensions MUST not alter the meaning of canonical fields.  

---

## 6. Adoption Checklist
An implementation is adoption-ready if:
1. Provides root regions (`^sys`, `^seq`, `^ah`).  
2. Implements canonical node types.  
3. Enforces deterministic invariants (ordering, TTL, sealing).  
4. Exposes selectors and diffs for debugging.  
5. Preserves IDs as opaque strings.  
6. Handles user-assigned types via canonical mapping.  
7. Exports snapshots with a spec version.  
8. Ignores unknown attributes without error.  

---

[← 06-debugging](06-debugging.md) | [↑ Spec Index](../README.md) | [→ 08-reference-implementations](08-reference-implementations.md)
