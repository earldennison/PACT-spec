[← 06-debugging](06-debugging.md) | [↑ Spec Index](../README.md) | [→ 08-reference-implementations](08-reference-implementations.md)

# 07 – Adoption and Compatibility

## 1. Purpose
This section provides guidance on **adopting PACT**, ensuring compatibility across implementations,  
and extending the model without breaking determinism or interoperability.

---

## 2. Incremental Adoption
### 2.1 Minimal Runtime
A minimal compliant PACT runtime MUST implement:
- Root regions (`^sys`, `^seq`, `^ah`) under `^root`.  
- Canonical node types (`mt`, `mc`, `cb`).  
- Deterministic sibling ordering.  
- TTL expiry and cascading cleanup.  
- Snapshot commit and serialization.  
- Basic selector support (`ctx.select`) and diffs (`ctx.diff`).

### 2.2 Incremental Layers
Implementations MAY adopt features in layers:
1. **Core structure** (tree + regions + turns).  
2. **Lifecycle** (TTL, expiry, sealing).  
3. **Optional pruning/compaction strategies** (informative; not required for conformance).  
4. **Selectors & diffs** (debugging, inspection).  

Each layer adds determinism and observability without invalidating earlier layers.

---

## 3. Compatibility Rules
### 3.1 Canonical Types
Implementations MUST always expose canonical types (`mt`, `mc`, `cb`),  
even if internal class names differ (e.g., `TextContextComponent` → `.cb`).

### 3.2 User-Assigned Types
Implementations MAY define user-assigned types (e.g., `cb:summary`).  
These MUST remain queryable via canonical selectors (`.cb`) and attributes.

### 3.3 Opaque IDs
Node `id` values are opaque. UUIDs, hashes, or other schemes MAY be used.  
All implementations MUST treat IDs as case-sensitive opaque strings for compatibility.

### 3.4 Extensions
Extensions (new attributes, roles, or node types) MUST NOT break invariants.  
They MUST use namespacing (`cb:foo`, `custom:bar`) to avoid collisions.  
Selectors MUST be able to target them via attributes.

---

## 4. Migration from Logs
### 4.1 Flat Logs
Flat conversation logs MAY be imported into PACT by mapping:
- `system` → `^sys .cb[role="system"]`  
- `user` → `^ah .mc .cb[role="user"]` (then sealed to `^seq`)  
- `assistant` → `^ah .mc .cb[role="assistant"]` (then sealed)  

Minimal example (import → seal):

```json
{
  "flat_log": [
    {"role": "system", "content": "You are helpful."},
    {"role": "user", "content": "Hello"},
    {"role": "assistant", "content": "Hi!"}
  ]
}
```

Imported into PACT prior to commit:

```json
{
  "root": {"children": [
    {"id": "sys-1", "nodeType": "^sys", "children": [
      {"id": "cb:sysA", "nodeType": "cb", "role": "system", "kind": "text", "offset": 0, "content": "You are helpful."}
    ]},
    {"id": "ah-1", "nodeType": "^ah", "children": [
      {"id": "mc:1", "nodeType": "mc", "offset": 0, "children": [
        {"id": "cb:u1", "nodeType": "cb", "role": "user", "kind": "text", "content": "Hello"},
        {"id": "cb:a1", "nodeType": "cb", "role": "assistant", "kind": "text", "content": "Hi!"}
      ]}
    ]}
  ]}
}
```

At commit, `^ah` is sealed into `^seq` as a new `mt`.

### 4.2 RAG and External Stores
External retrieval-augmented memory MAY insert items as `cb` nodes.  
These nodes MUST respect TTL, pruning, and canonical ordering.

Example (RAG insert with TTL):

```json
{
  "id": "cb:rag1",
  "nodeType": "cb",
  "role": "system",
  "kind": "text",
  "content": "Doc: How to reset a password...",
  "ttl": 2,
  "offset": 1
}
```

Placement guidance:
- Attach RAG `cb` under the relevant turn’s post‑context (`offset > 0`) or into `^ah` during the current cycle.
- Use `ttl` to expire retrievals deterministically; pruning SHALL follow priority → age → id.

---

## 5. Versioning
### 5.1 Spec Version
Snapshots SHOULD include a `spec_version` field (e.g., `"PACT/0.1"`).  
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
