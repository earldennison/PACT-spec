[← 01-architecture](01-architecture.md) | [↑ Spec Index](../README.md) | [→ 03-lifecycle-ttl](03-lifecycle-ttl.md)

# 02 – Invariants

## 1. Purpose
This section defines the **normative invariants** that every PACT implementation MUST satisfy.  
These invariants guarantee deterministic structure, placement, lifecycle, and rendering.

## 2. Tree and Regions
### 2.1 Single Root
The context is a single rooted tree. The root container (`^root`) MUST exist and MUST NOT be pruned or expired.

### 2.2 Required Regions
Under `^root`, implementations MUST provide exactly these regions:
- `^sys` — System Header (persistent context)
- `^seq` — Sealed sequence of past turns
- `^ah` — Active Head (current turn; at most one)

### 2.3 Region Movement (Pre‑Commit)
Prior to commit (within a cycle), implementations MAY re‑parent any node across regions except:
- `mt` (message turns) — MAY move only via sealing transition `^ah` → `^seq`;
- the region container `^sys` — MUST NOT be re‑parented.

At commit, region placement is evaluated against invariants. Between snapshots, re‑parenting is observable via diffs (§9.3). Implementations SHOULD realize cross‑region relocations by add/remove operations that yield identical rendered bytes at commit.  
Note: Cross‑region movement is observationally equivalent to remove+add, not an in‑place mutation; in‑place region changes MUST NOT be observable across snapshots.

## 3. Node Identity and Headers
### 3.1 Stable ID
Each node MUST have a stable `id` (hash or UUID). The `id` MUST NOT change during the node's lifetime within a snapshot. Implementations MAY assign new IDs during internal rematerialization as long as final snapshots are byte-stable.

### 3.2 Required Headers
Each node MUST include:  
`id`, `nodeType`, `offset`, `ttl`, `priority`, `cycle`, `created_at_ns`, `created_at_iso`, `creation_index`.

#### 3.2.1 Attribute Stability Table (Normative)

| Field            | Type     | MUST/MAY | Stability (Immutability)                 | Notes |
|------------------|----------|----------|------------------------------------------|-------|
| `id`             | string   | MUST     | Immutable within a snapshot               | Opaque identifier (UUID/hash) |
| `nodeType`       | string   | MUST     | Immutable                                 | Canonical or namespaced user type |
| `offset`         | number   | MUST     | Immutable post-creation within a cycle    | `<0` pre, `0` core (`mc`), `>0` post |
| `ttl`            | number∣null | MUST  | MAY change by lifecycle evaluation        | Cycle-based TTL semantics |
| `priority`       | number   | MUST     | Immutable unless policy updates           | Used for pruning order |
| `cycle`          | number   | MUST     | Immutable                                 | Introducing commit’s cycle |
| `created_at_ns`  | number   | MUST     | Immutable                                 | Monotonic timestamp |
| `created_at_iso` | string   | MUST     | Immutable                                 | ISO8601 mirror of timestamp |
| `creation_index` | number   | MUST     | Immutable                                 | Per-cycle tie-breaker |
| `role`           | string   | MAY      | Immutable                                 | e.g., `user`, `assistant`, `tool`, `system` |
| `kind`           | string   | MAY      | Immutable                                 | e.g., `text`, `call`, `result`, `image` |
| `content_hash`   | string   | MAY      | MAY change when content changes           | Excluded from ordering |
| Custom `data_*`  | any      | MAY      | SHOULD remain stable per meaning          | Namespaced; see §3.5 |
| Custom `content_*` | any    | MAY      | MAY change with content                   | Namespaced; see §3.5 |

### 3.3 Canonical Types
Canonical classes:
- `mt` — MessageTurn
- `mc` — MessageContainer (offset=0)
- `cb` — ContentBlock

### 3.4 User-Assigned Types
Implementations MAY define user types (e.g., `cb:summary`). Selectors MUST still resolve these as `.cb`.

### 3.5 Attribute Creation and Stability (Normative)
- Source of truth: Selector attributes map to node headers. Implementations MUST populate all mandatory headers; optional headers MAY be omitted.
- Namespacing: Custom attributes MUST be namespaced (e.g., `content_*`, `data_*`, `cb:foo`) and MUST NOT collide with reserved names.
- Types: Header types are stable: `offset` | `ttl` | `priority` | `cycle` | `created_at_ns` | `creation_index` are numeric; `id` | `nodeType` | `created_at_iso` | `role` | `kind` are strings. Custom attributes SHOULD use a single stable type across a node’s lifetime.
- Immutability: Creation-time headers MUST remain stable; only lifecycle fields (e.g., `ttl`) MAY change per §3 and §5.
- Defaults and nulls: Missing optional attributes compare as `null` per Selectors §5.5; required headers MUST be present (non-null).
- Serialization: Attributes MUST serialize canonically and deterministically; unknown attributes MUST NOT affect canonical ordering.
- Reserved names: Implementations MUST NOT redefine `id`, `nodeType`, `offset`, `ttl`, `priority`, `cycle`, `created_at_ns`, `created_at_iso`, `creation_index`, `content_hash`.

## 4. Placement and Ordering
This section is the normative, canonical source for placement (offset semantics and core placement) and sibling ordering. Other chapters MUST reference this section and MUST NOT redefine these rules.
### 4.1 Offsets
Offsets define placement:
- `<0` → pre-context
- `=0` → core (`mc`)
- `>0` → post-context

#### 4.1.1 Example: Content in offset nodes

A turn (`mt`) with content-bearing nodes at negative and positive offsets, and the core message in `mc @ offset=0`:

```json
{
  "id": "mt:42",
  "nodeType": "mt",
  "children": [
    { "id": "cb:pre1",  "nodeType": "cb", "offset": -1, "role": "system",   "kind": "text",   "content": "Contextual hint" },
    { "id": "mc:42",    "nodeType": "mc", "offset": 0,  "children": [
      { "id": "cb:u42", "nodeType": "cb",               "role": "user",     "kind": "text",   "content": "Hello" }
    ] },
    { "id": "cb:post1", "nodeType": "cb", "offset": 1,  "role": "tool",     "kind": "result", "content": "status: ok" }
  ]
}
```

### 4.2 Core Container
Each turn MUST have exactly one `mc` at `offset=0`.

### 4.3 Ordering
Siblings MUST be ordered by this canonical sequence:
1. `offset` ascending,
2. `created_at_ns` ascending,
3. `creation_index` ascending,
4. `id` lexicographic.

### 4.4 Monotonicity Guarantees
Implementations MUST ensure:
- **Within a single cycle**: `created_at_ns` values MUST be strictly monotonic (no duplicates)
- **Across cycles**: `created_at_ns` MAY reset but SHOULD continue monotonically when possible
- **Creation index**: MUST increment monotonically within each cycle (0, 1, 2, ...)
- **Tie-breaking**: `creation_index` resolves identical timestamps within the same cycle

**Example ordering:**
```
Node A: offset=-1, created_at_ns=1000, creation_index=0, id="a"
Node B: offset=-1, created_at_ns=1000, creation_index=1, id="z"  
Node C: offset=0,  created_at_ns=1001, creation_index=2, id="b"
→ Final order: A, B, C (B comes before C despite later timestamp)
```

**Conformance**: Implementations MAY employ any internal sorting, provided the externally observable order equals the canonical order (offset asc → created_at_ns asc → creation_index asc → id lexicographic).

### 4.5 Re-Parenting

Prior to commit, nodes MAY be re‑parented (moved to a different parent) provided that:
1. `mt` nodes themselves are not re‑parented (turn identity and order are fixed; `mt` moves only by sealing `^ah` → `^seq`).
2. The region container `^sys` is not re‑parented.
3. Conformance is evaluated on snapshot structure and serialized bytes, not internal mutation mechanics.

After sealing, `mc@0` cores are immutable (§6.4). Non‑core nodes MAY be attached or detached relative to sealed turns by adding/removing nodes (e.g., post‑context attachments); sealed records are not mutated in place.

## 5. Lifecycle
### 5.1 TTL
`ttl` is cycle-based:
- `None` → persistent
- `0` → expires next commit
- `N` → expires after N cycles

### 5.2 Expiry Timing
Expiry is evaluated at commit. Expired nodes MUST be removed before pruning.

### 5.3 Cascading Cleanup
Empty containers marked removable MUST also be removed. Root and region containers MUST persist.

#### 5.3.1 Removable Containers (Normative)
- Definition: A container is "removable" if and only if it is explicitly marked `removable=true` at creation time or by a deterministic policy defined in configuration (e.g., ephemeral post-context groups). The root (`^root`) and region containers (`^sys`, `^seq`, `^ah`) MUST NOT be removable.
- Stability: The `removable` flag MUST be immutable across the container’s lifetime. Implementations MUST NOT toggle the `removable` flag after creation.
- Cascade: When all children of a removable container are expired or pruned, the container MUST be removed in the same commit.

### 5.4 Reference Safety
Nodes with live references MUST NOT be removed (expired or pruned). If liveness cannot be determined, implementations MUST preserve such nodes conservatively and MAY defer expiry across commits to satisfy this requirement.

## 6. Turns and Sealing
### 6.1 Active Head
There MUST be at most one `^ah` per snapshot.

### 6.2 Sealing
At commit, `^ah` is sealed into `^seq` as a new `mt`. The sealed `mc` is immutable.

### 6.3 Depth
In `^seq`, depth counts newest→oldest.  
`depth=1` = most recent sealed turn.  
`^ah` is not depth-addressable.

### 6.4 Immutability
Sealed cores (`mc @ offset=0`) MUST NOT be mutated. Snapshots are immutable records. Non‑core nodes (pre/post context, summaries, redactions, corrections, containers) MAY be added or removed in subsequent cycles, including attachments targeting sealed turns, via new nodes and lifecycle—never by mutating sealed bytes in place.

## 7. Pruning (Optional)
**Preface.** Pruning is **optional**. If pruning is disabled, an implementation remains conformant by
skipping this section entirely. When pruning **is** implemented, it MUST follow the canonical order and
determinism requirements below.
### 7.1 Order
After TTL expiry, implementations MAY apply pruning or compaction policies. When pruning is applied, nodes MUST be pruned by `priority`, then age, then `id`.

### 7.2 Protection
Recent K turns, pinned nodes, and references SHOULD be protected.

### 7.3 Reproducibility
Given the same pre-state and the same declared pruning/compaction policies and limits, all conformant implementations MUST yield identical post-state. Determinism is judged on the resulting snapshot structure and serialized bytes.

Note: How an implementation declares or configures policies (e.g., token, node, depth, region) is out of scope for this specification.

### 7.4 Budgeting (Out of Scope)
Budgeting strategies (e.g., token budgets, node/depth caps, heuristic scoring)
are **non-normative**. This specification only defines the **order of pruning** when pruning is enabled.
Runtime-specific budgeting MAY influence what gets pruned, but MUST NOT violate canonical order or other invariants.

## 8. Snapshots and Serialization
### 8.1 Atomic Commit
Each cycle produces a snapshot. Traversal MUST be deterministic.

### 8.2 Serialization
Rendering MUST follow canonical order. Identical snapshots → identical bytes.

### 8.3 No Side Effects
Serialization MUST NOT mutate the tree.

### 8.4 Snapshot Addressing
Snapshots are addressable:
- `@t0` current
- `@t-1` previous
- `@cN` absolute cycle

### 8.5 Snapshot Boundary (Normative)
See Lifecycle §7 for Commit Sequence (Normative). A snapshot MUST serialize to the provider‑bound bytes for that cycle. Persistence MAY be asynchronous; equivalence is judged on logical content and rendered bytes.

## 9. Selectors and Diffs
### 9.1 Purity
`ctx.select` MUST be pure. Same snapshot + selector → same node IDs.

### 9.2 Features
Selectors MUST support:
- Roots: `^sys`, `^seq`, `^ah`, `^root`
- Types: `.mt`, `.mc`, `.cb`
- IDs: `#<id>`
- Pseudos: `:pre`, `:core`, `:post`, `:depth(n)`, `:first`, `:last`, `:nth(n)`
- Attributes: `[offset] [ttl] [priority] [cycle] [nodeType] [role] [kind]`
- Combinators: descendant (`A B`), child (`A > B`)

### 9.3 Diffs
`ctx.diff(A,B,selector)` MUST detect changes by node `id` and report both content and structural changes. At minimum, diffs SHOULD include:
- Component lifecycle: `components_added`, `components_removed`, field‑wise `components_modified` (including `ttl` changes and header changes per §3.5);
- Structural moves (re‑parenting):
  - In the old parent: `children_removed += <child_id>`
  - In the new parent: `children_added += <child_id>`
  - OPTIONAL on the child: `parent_changed {previous_parent_id, new_parent_id}`
TTL changes MUST be reported as modifications.
Implementations MAY provide richer change models, but MUST preserve determinism and id‑based correspondence.

## 10. Validation
- Only one `mc` at offset=0 per turn.  
- Only one `^ah` per snapshot.  
- Unknown `nodeType` values MUST fall back to canonical mapping.  
- Invalid selectors MUST raise errors.

## 11. Conformance Checklist
An implementation is conformant if:
1. Regions `^sys`, `^seq`, `^ah` exist.
2. Nodes carry required headers.
3. Each turn has exactly one `mc@0`.
4. Ordering rules are enforced.
5. TTL expiry + cascade occur at commit.
6. Sealed cores are immutable; snapshots are immutable.
7. If implemented, pruning policies are deterministic (priority → age → id).
8. Snapshots serialize to stable bytes.
9. Selectors are pure and complete.
10. Diffs compare by `id`.
11. Invalid placements are rejected.

---

## Example

```text
mt (MessageTurn)
 ├─ cb (offset <0)   ← pre-context
 ├─ mc (offset =0)   ← core
 │    └─ cb          ← content
 └─ cb (offset >0)   ← post-context
```
---

## 12. Provider Thread Mapping

### 12.1 Purpose
This section defines how the PACT tree is rendered into a **linear provider thread**.  
Given the same snapshot, all implementations MUST produce identical serialized context bytes.

### 12.2 Regions in Render Order
1. **System Header (`^sys`)** — rendered first.  
2. **Sealed Sequence (`^seq`)** — rendered in order, oldest → newest.  
3. **Active Head (`^ah`)** — rendered last (in-progress content).

### 12.3 Turn Layout
Within each `mt` (or `^ah`):
1. Pre-context (`offset < 0`) — all `cb` children.  
2. Core container (`mc @ offset=0`) — serialized as the main message body.  
3. Post-context (`offset > 0`) — all `cb` children.

### 12.4 Roles
- Each `cb` MAY carry a `role` (e.g., `"user"`, `"assistant"`, `"tool"`, `"system"`).  
- Roles MUST be preserved and mapped consistently to provider schemas.  
- If no role is given, default: `"system"` for `^sys`, `"user"` for turns.

### 12.5 Serialization Units
Each `cb` serializes to an object:
```json
{
  "id": "cb:u1",
  "role": "user",
  "kind": "text",
  "content": "Hello!"
}
```

- id MUST be preserved for diffs.

- kind SHOULD indicate the modality ("text", "call", "result", etc.).

### 12.6 Ordering

- Within turns: canonical order per §4.3.

- Across turns: ^seq (oldest→newest), then ^ah.

### 12.7 Conformance

1. An implementation is conformant if:

2. Renders ^sys → ^seq → ^ah.

3. Preserves canonical sibling order.

4. Preserves roles and IDs.

5. Produces identical output for identical snapshots.

6. Serialization is side-effect free.



[← 01-architecture](01-architecture.md) | [↑ Spec Index](../README.md) | [→ 03-lifecycle-ttl](03-lifecycle-ttl.md)

---

### 12.8 End-to-End Provider Thread Example (Normative)

Given this example snapshot structure at `@t0`:

```json
{
  "root": {
    "id": "root-1",
    "children": [
      {
        "id": "sys-1",
        "nodeType": "^sys",
        "children": [
          {"id": "cb:sysA", "role": "system", "kind": "text", "offset": 0, "content": "You are a helpful assistant."}
        ]
      },
      {
        "id": "seq-1",
        "nodeType": "^seq",
        "children": [
          {
            "id": "mt:1",
            "nodeType": "mt",
            "children": [
              {"id": "cb:u1", "role": "user", "kind": "text", "offset": 0, "content": "Hello"}
            ]
          },
          {
            "id": "mt:2",
            "nodeType": "mt",
            "children": [
              {"id": "cb:a1", "role": "assistant", "kind": "text", "offset": 0, "content": "Hi! How can I help?"}
            ]
          }
        ]
      },
      {
        "id": "ah-1",
        "nodeType": "^ah",
        "children": [
          {"id": "cb:u2", "role": "user", "kind": "text", "offset": 0, "content": "Summarize the above."}
        ]
      }
    ]
  }
}
```

The provider thread serialization MUST be in this exact order (regions: `^sys` → `^seq` oldest→newest → `^ah`; within each, canonical sibling order per §4.3):

```json
[
  {"id": "cb:sysA", "role": "system", "kind": "text", "content": "You are a helpful assistant."},
  {"id": "cb:u1",  "role": "user",   "kind": "text", "content": "Hello"},
  {"id": "cb:a1",  "role": "assistant", "kind": "text", "content": "Hi! How can I help?"},
  {"id": "cb:u2",  "role": "user",   "kind": "text", "content": "Summarize the above."}
]
```

Independent implementations MUST reproduce these bytes for the given snapshot.

### 12.9 Provider Thread Example with Pre/Post Context (Normative)

Given this example snapshot structure at `@t0` containing pre-context (offset < 0) and post-context (offset > 0):

```json
{
  "root": {
    "id": "root-2",
    "children": [
      {
        "id": "sys-2",
        "nodeType": "^sys",
        "children": [
          {"id": "cb:sysB", "role": "system", "kind": "text", "offset": 0, "content": "System header B"}
        ]
      },
      {
        "id": "seq-2",
        "nodeType": "^seq",
        "children": [
          {
            "id": "mt:10",
            "nodeType": "mt",
            "children": [
              {"id": "cb:pre1",  "role": "system",   "kind": "text",   "offset": -1, "content": "Pre-context hint"},
              {"id": "cb:core1", "role": "user",     "kind": "text",   "offset": 0,  "content": "Hello with context"},
              {"id": "cb:post1", "role": "tool",     "kind": "result", "offset": 1,  "content": "status: ok"}
            ]
          }
        ]
      },
      {
        "id": "ah-2",
        "nodeType": "^ah",
        "children": [
          {"id": "cb:pre2",  "role": "system",   "kind": "text",   "offset": -1, "content": "AH pre"},
          {"id": "cb:core2", "role": "user",     "kind": "text",   "offset": 0,  "content": "Working..."},
          {"id": "cb:post2", "role": "assistant", "kind": "text",   "offset": 1,  "content": "Interim note"}
        ]
      }
    ]
  }
}
```

The provider thread serialization MUST be in this exact order (regions: `^sys` → `^seq` oldest→newest → `^ah`; within each, canonical sibling order per §4.3, i.e., pre < core < post):

```json
[
  {"id": "cb:sysB",  "role": "system",   "kind": "text",   "content": "System header B"},
  {"id": "cb:pre1",  "role": "system",   "kind": "text",   "content": "Pre-context hint"},
  {"id": "cb:core1", "role": "user",     "kind": "text",   "content": "Hello with context"},
  {"id": "cb:post1", "role": "tool",     "kind": "result", "content": "status: ok"},
  {"id": "cb:pre2",  "role": "system",   "kind": "text",   "content": "AH pre"},
  {"id": "cb:core2", "role": "user",     "kind": "text",   "content": "Working..."},
  {"id": "cb:post2", "role": "assistant", "kind": "text",   "content": "Interim note"}
]
```

Independent implementations MUST reproduce these bytes for the given snapshot.

[← 01-architecture](01-architecture.md) | [↑ Spec Index](../README.md) | [→ 03-lifecycle-ttl](03-lifecycle-ttl.md)