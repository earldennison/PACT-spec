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
- `seg` (message turns) — MAY move only via sealing transition `^ah` → `^seq`;
- the region container `^sys` — MUST NOT be re‑parented.

At commit, region placement is evaluated against invariants. Between snapshots, re‑parenting is observable via diffs (§9.3). Implementations SHOULD realize cross‑region relocations by add/remove operations that yield identical rendered bytes at commit.  
Note: Cross‑region movement is observationally equivalent to remove+add, not an in‑place mutation; in‑place region changes MUST NOT be observable across snapshots.

### 2.4 Global Depth Axis (Normative)

There is one global integer axis `depth = d ∈ ℤ` that addresses all top-level containers (within a fixed snapshot):

- `d = 0` → the Active Turn container (the in-progress turn)
- `d ≥ 1` → sealed turns in commit order within the snapshot; `d = 1` is the most recent sealed turn in that snapshot, `d = 2` the one before it, etc.
- `d = -1` → the System container (implementation-/provider-owned)
- (Reserved) `d ≤ -2` → reserved for future system strata; MUST NOT be used by userland without explicit extension.

### 2.5 Region Aliases (Normative)

The symbols `^ah`, `^seq`, and `^sys` are **pure aliases** over `depth`:

- `^ah` ≡ `depth(0)`
- `^seq` ≡ the set `{ depth(n) | n ≥ 1 }`
- `^sys` ≡ `depth(-1)`

These aliases are **syntactic sugar only**. They MUST NOT confer capabilities, mutability, or permissions different from their `depth(...)` equivalents. Depth is structural; cross‑snapshot time addressing uses `@t…`.

### 2.6 Equivalence Law (Normative)

For any selector `S`, replacing:
- `^ah` with `depth(0)`,
- `^sys` with `depth(-1)`

MUST select the identical node(s). History selection uses `@t…` and is not encoded by depth.


## 3. Node Identity and Headers
### 3.1 Stable ID (Global Uniqueness)
Each node MUST have a stable `id` (hash or UUID). The `id` MUST NOT change during the node's lifetime within a snapshot, and MUST be globally unique across the addressable history (no reuse across live snapshots). Implementations MAY assign new IDs during internal rematerialization as long as final snapshots are byte-stable.

### 3.1.1 Normative Rule — `parent_id`
Every node MUST have a `parent_id` that references its immediate parent’s `id`, except designated root nodes, which MUST have `parent_id = null`.

“Root” means any top‑level container defined by the implementation (e.g., the episode’s top‑level containers). Roots are the only nodes allowed to have `parent_id = null`.

### 3.2 Required Headers
Each node MUST include:  
`id`, `nodeType`, `offset`, `ttl`, `priority`, `cycle`, `created_at_ns`, `created_at_iso`, `creation_index`.

#### 3.2.a Structural headers at commit (t0)
At the snapshot boundary (`@t0`), each node’s structural headers are defined as:

- id: string — provider-/system-unique (no reuse across live snapshots)
- nodeType: enum {`seg`, `cont`, `block`, ...}
- parent_id: string | null — immediate parent container’s `id` at commit time; roots use `null` (only top‑level containers are roots)
- offset: int — position relative to the parent’s ordered children
- ttl: number | null — lifecycle in episodes (see 03 – Lifecycle §3.1)
- created_at_ns: uint64 — monotonic timestamp
- creation_index: uint — tie‑breaker within the same timestamp window

Notes:
- `cad` (cadence) is a positive integer when present; default behavior is equivalent to `cad=1` if omitted (see 03 – Lifecycle §3.4). Implementations MAY store this under the header name `cad`.
- `parent_id` is a machine header for determinism and diffs. It need not be a user‑addressable selector primitive and MAY be omitted from human‑friendly renders when it can be derived unambiguously from the tree shape.

#### 3.2.b Structural header constraints (Normative)
parent_id ⇔ root:
- `parent_id = null` ⇔ node is a root
- `parent_id ≠ null` ⇔ node is NOT a root

Parent existence and capability:
- The referenced parent MUST exist in the same commit/episode and MUST be container‑capable (may hold children).

No cycles:
- A node MUST NOT be its own parent.
- A node MUST NOT be an ancestor of its parent.

Offset frame and determinism:
1) `parent_id` defines the reference frame for `offset`. Changing `parent_id` implies recomputing `offset` in the new parent’s frame.
2) Ordering is governed by the parent’s canonical child‑ordering invariants (offset ↑, created_at_ns ↑, creation_index ↑, id ↑). `parent_id` does not alter ordering rules; it only names the frame.

Instances and projections:
3) Cloned/projection instances (from cadence/TTL overlap) each have distinct `id` values and a `parent_id` consistent with their realized placement.

Selector interface:
4) Selector semantics remain structural; authoring is not required to use IDs. `parent_id` is a machine header for determinism and diffs, not a user‑facing selector primitive.

Schema hint (non‑normative):
```yaml
parent_id:
  type: [string, "null"]
  description: "Immediate parent’s id; null only for top‑level roots."
  required: true
```

#### 3.2.1 Attribute Stability Table (Normative)

| Field            | Type     | MUST/MAY | Stability (Immutability)                 | Notes |
|------------------|----------|----------|------------------------------------------|-------|
| `id`             | string   | MUST     | Immutable within a snapshot               | Opaque identifier (UUID/hash) |
| `nodeType`       | string   | MUST     | Immutable                                 | Canonical or namespaced user type |
| `parent_id`      | string∣null | MUST  | Write‑protected (changes only via re‑parent) | Machine header; not a selector primitive |
| `offset`         | number   | MUST     | Immutable post-creation within a cycle    | `<0` pre, `0` core (`cont`), `>0` post |
| `ttl`            | number∣null | MUST  | MAY change by lifecycle evaluation        | Cycle-based TTL semantics |
| `priority`       | number   | MUST     | Immutable unless policy updates           | Used for pruning order |
| `cycle`          | number   | MUST     | Immutable                                 | Introducing commit’s cycle |
| `created_at_ns`  | number   | MUST     | Immutable                                 | Monotonic timestamp |
| `created_at_iso` | string   | MUST     | Immutable                                 | ISO8601 mirror of timestamp |
| `creation_index` | number   | MUST     | Immutable                                 | Per-cycle tie-breaker |
| `content_hash`   | string   | MAY      | MAY change when content changes           | Excluded from ordering |
| Custom `data_*`  | any      | MAY      | SHOULD remain stable per meaning          | Namespaced; see §3.5 |
| Custom `content_*` | any    | MAY      | MAY change with content                   | Namespaced; see §3.5 |

### 3.3 Canonical Types
Canonical classes:
- `seg` — MessageTurn
- `cont` — Container (offset=0)
- `block` — Block

### 3.4 User-Assigned Types
Implementations MAY define user types (e.g., `summary`). Selectors MUST still resolve these as `.block`.

### 3.5 Attribute Creation and Stability (Normative)
- Source of truth: Selector attributes map to node headers. Implementations MUST populate all mandatory headers; optional headers MAY be omitted.
- Namespacing: Custom attributes MUST be namespaced (e.g., `content_*`, `data_*`, `block:foo`) and MUST NOT collide with reserved names.
- Types: Header types are stable: `offset` | `ttl` | `priority` | `cycle` | `created_at_ns` | `creation_index` are numeric; `id` | `nodeType` | `created_at_iso` are strings. Custom attributes SHOULD use a single stable type across a node’s lifetime.
- Immutability: Creation-time headers MUST remain stable; only lifecycle fields (e.g., `ttl`) MAY change per §3 and §5.
- Defaults and nulls: Missing optional attributes compare as `null` per Selectors §5.5; required headers MUST be present (non-null).
- Serialization: Attributes MUST serialize canonically and deterministically; unknown attributes MUST NOT affect canonical ordering.
- Reserved names: Implementations MUST NOT redefine `id`, `nodeType`, `offset`, `ttl`, `priority`, `cycle`, `created_at_ns`, `created_at_iso`, `creation_index`, `content_hash`.

### 3.6 Keys, Overlap, and ODI (Normative)
- **Key multiplicity**: The same `key` attribute MAY appear on multiple live nodes within a single snapshot (temporal overlap from cadence and TTL).
- **Overlap Demotion Invariant (ODI)**: If a new instance with `key=K` is materialized while a prior `key=K` instance is still alive, placement MUST satisfy `depth(prev_id) > depth(new_id)` for all prior live instances, under Universal Depth (03 – Lifecycle §3.6). Engines MUST either auto‑demote prior instances to satisfy ODI or reject the placement with an error (e.g., `OverlapWouldViolateODI`).
- **Determinism (per‑key selection and rendering of same‑key sets)**: When selecting among multiple instances with the same `key` without an explicit ordering directive, the canonical order MUST be: primary `depth` descending, secondary `created_at_ns` descending, then `creation_index` descending, and finally `id` ascending as a stable tie‑breaker.

## 4. Placement and Ordering
This section is the normative, canonical source for placement (offset semantics and core placement) and sibling ordering. Other chapters MUST reference this section and MUST NOT redefine these rules.
### 4.1 Offsets
Offsets define placement:
- `<0` → pre-context
- `=0` → core (`cont`)
- `>0` → post-context

#### 4.1.1 Example: Content in offset nodes

A segment (`seg`) with content-bearing nodes at negative and positive offsets, and the core message in `cont @ offset=0`:

```json
{
  "id": "seg:42",
  "nodeType": "seg",
  "children": [
    { "id": "block:pre1",  "nodeType": "block", "offset": -1,   "content": "Contextual hint" },
    { "id": "cont:42",    "nodeType": "cont", "offset": 0,  "children": [
      { "id": "block:u42", "nodeType": "block",                "content": "Hello" }
    ] },
    { "id": "block:post1", "nodeType": "block", "offset": 1,    "content": "status: ok" }
  ]
}
```

### 4.2 Core Container
Each segment MUST have exactly one `cont` at `offset=0`.

### 4.3 Ordering
Siblings MUST be ordered by this canonical sequence:
1. `offset` ascending,
2. `created_at_ns` ascending,
3. `creation_index` ascending,
4. `id` lexicographic.

Note: Canonical sibling ordering governs structural traversal and rendering. Per‑key selection determinism (see §3.6) governs how to order or choose among multiple overlapping instances of the same logical `key`.

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
1. `seg` nodes themselves are not re‑parented (segment identity and order are fixed; `seg` moves only by sealing `^ah` → `^seq`).
2. The region container `^sys` is not re‑parented.
3. Conformance is evaluated on snapshot structure and serialized bytes, not internal mutation mechanics.

After sealing, `cont[offset=0]` cores are immutable (§6.4). Non‑core nodes MAY be attached or detached relative to historical turns by adding/removing nodes (e.g., post‑context attachments); sealed records are not mutated in place.

#### 4.5.1 Operation: move(node, to_parent, to_offset)
Preconditions:
- `to_parent` exists and is container‑capable; `node` is not an ancestor of `to_parent`.

Steps:
1) Detach `node` from `parent_id`’s children (preserve sibling order of the remaining children).
2) Insert `node` into `to_parent.children` at `to_offset` (or the normalized end if out‑of‑bounds).
3) Set `node.parent_id := to_parent.id`; recompute `node.offset` in the new parent’s frame per §4.3.

Diff emission (minimum):
- `parent_changed: { previous_parent_id, new_parent_id }`
- `offset_changed: { previous_offset, new_offset }`

Errors:
- `INVALID_PARENT` — `to_parent` missing or not in the same commit/episode
- `PARENT_NOT_CONTAINER` — `to_parent` is not container‑capable
- `CYCLE_DETECTED` — operation would create a cycle (e.g., `node` becomes ancestor/descendant of itself)

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
Reference Safety (normative position)

Normative: PACT intentionally remains silent about application-level policies for pruning, garbage collection, expiry, retention, or any lifecycle policy that removes or archives nodes from a runtime. PACT specifies structural invariants, addressing, ordering, offsets, and snapshot/commit semantics — it does not prescribe, require, or prohibit any node deletion or retention strategies. Implementations and applications are free to adopt their own retention, pruning, or archival policies; any such policies are outside the scope of this specification.

Note (non-normative): Runtimes MAY provide helper APIs for consumers (for example, to inspect references or enumerate child nodes) as implementation conveniences. Such APIs are explicitly implementation-specific and non-normative and must not be interpreted as part of PACT's normative conformance expectations.

## 6. Sealing and Context States
### 6.1 Active Head
There MUST be at most one `^ah` per snapshot (structural anchor for the working set).

### 6.2 Sealing
At commit, the context is sealed into a snapshot (sealed context). `@t0` is never sealed. Sealed snapshots are addressed by `@t-1`, `@t-2`, …

Fresh-@t0 equivalence (Normative): If no writes occur after the latest commit, `@t0 ≡ @t-1`. Any write breaks the equivalence; the next commit materializes a new sealed snapshot at `@t-1`.

### 6.3 Depth
Depth is structural only (see §2.4). It MUST NEVER encode time. History addressing uses `@t…`.

### 6.4 Immutability
Snapshots are immutable records. Non‑sealed working content MAY be added or removed in subsequent cycles via new nodes and lifecycle—never by mutating sealed bytes in place.

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

### 8.4 Time Prefix Addressing
Time prefixes are addressable:
- `@t0` active (unsealed) working set (default if omitted)
- `@t-1` newest sealed snapshot; `@t-2`, … older sealed snapshots
- `@cN` absolute cycle

### 8.5 Snapshot Boundary (Normative)
See Lifecycle §7 for Commit Sequence (Normative). A snapshot MUST serialize to the provider‑bound bytes for that cycle. Persistence MAY be asynchronous; equivalence is judged on logical content and rendered bytes.

## 9. Selectors and Diffs
### 9.1 Purity
`ctx.select` MUST be pure. Same snapshot + selector → same node IDs.

### 9.2 Features
Selectors MUST support:
- Roots: `^sys`, `^seq`, `^ah`, `^root`
- Types: `.seg`, `.cont`, `.block`
- IDs: `{ id="…" }`
- Predicates: `:pre`, `:core`, `:post`, `:first`, `:last`, `:nth(n)`
- Attributes: `[offset] [ttl] [cad] [priority] [cycle] [nodeType]`
- Structural hops: descendant (`A B`), child (`A > B`)

### 9.3 Diffs
`ctx.diff(A,B,selector)` MUST detect changes by node `id` and report both content and structural changes. At minimum, diffs SHOULD include:
- Lifecycle changes: `added`, `removed`, and field‑wise `changed` (including `ttl` changes and header changes per §3.5);
- Structural moves (re‑parenting):
  - In the old parent: `children_removed += <child_id>`
  - In the new parent: `children_added += <child_id>`
  - OPTIONAL on the child: `parent_changed {previous_parent_id, new_parent_id}`
TTL changes MUST be reported as modifications.
Implementations MAY provide richer change models, but MUST preserve determinism and id‑based correspondence.

## 10. Validation
- Only one `cont` at offset=0 per segment.  
- Only one `^ah` per snapshot.  
- Unknown `nodeType` values MUST fall back to canonical mapping.  
- Invalid selectors MUST raise errors.

## 11. Conformance Checklist
An implementation is conformant if:
1. Regions `^sys`, `^seq`, `^ah` exist.
2. Nodes carry required headers.
3. Each segment has exactly one `cont[offset=0]`.
4. Ordering rules are enforced.
5. TTL expiry + cascade occur at commit.
6. Sealed cores are immutable; snapshots are immutable.
7. If implemented, pruning policies are deterministic (priority → age → id).
8. Snapshots serialize to stable bytes.
9. Selectors are pure and complete.
10. Diffs compare by `id`.
11. Invalid placements are rejected.

### No Active-Cycle Immutability (Normative)
- Conformant implementations MUST allow edits at `depth ≤ 0` during the active cycle, subject to invariants and authorization. Any blanket “immutable at t0/at/`^sys`/`^ah`” behavior is non‑conformant.

---

## Example

```text
seg (MessageTurn)
 ├─ block (offset <0)   ← pre-context
 ├─ cont (offset =0)   ← core
 │    └─ block          ← content
 └─ block (offset >0)   ← post-context
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
Within each `seg` (or `^ah`):
1. Pre-context (`offset < 0`) — all `block` children.  
2. Core container (`cont @ offset=0`) — serialized as the main message body.  
3. Post-context (`offset > 0`) — all `block` children.

### 12.4 Attributes
This specification does not define or require a `role` attribute. Attributes used in examples are limited to structural/neutral fields.

### 12.5 Serialization Units
Each `block` serializes to an object:
```json
{
  "id": "block:u1",
  "content": "Hello!"
}
```

- id MUST be preserved for diffs.

### 12.6 Ordering

- Within turns: canonical order per §4.3.

- Across turns: ^seq (oldest→newest), then ^ah.

### 12.7 Conformance

1. An implementation is conformant if:

2. Renders ^sys → ^seq → ^ah.

3. Preserves canonical sibling order.

4. Preserves IDs.

5. Produces identical output for identical snapshots.

6. Serialization is side-effect free.



[← 01-architecture](01-architecture.md) | [↑ Spec Index](../README.md) | [→ 03-lifecycle-ttl](03-lifecycle-ttl.md)

---

### 12.8 End-to-End Provider Thread Example (Normative)

Given this example snapshot structure at `@t0`:

```json
{
  "spec_version": "PACT/1.0.0",
  "root": {
    "id": "root-1",
    "children": [
      {
        "id": "sys-1",
        "nodeType": "^sys",
        "children": [
          {"id": "block:sysA", "offset": 0, "content": "You are a helpful assistant."}
        ]
      },
      {
        "id": "seq-1",
        "nodeType": "^seq",
        "children": [
          {
            "id": "seg:1",
            "nodeType": "seg",
            "children": [
              {"id": "block:u1", "offset": 0, "content": "Hello"}
            ]
          },
          {
            "id": "seg:2",
            "nodeType": "seg",
            "children": [
              {"id": "block:a1", "offset": 0, "content": "Hi! How can I help?"}
            ]
          }
        ]
      },
      {
        "id": "ah-1",
        "nodeType": "^ah",
        "children": [
          {"id": "block:u2", "offset": 0, "content": "Summarize the above."}
        ]
      }
    ]
  }
}
```

The provider thread serialization MUST be in this exact order (regions: `^sys` → `^seq` oldest→newest → `^ah`; within each, canonical sibling order per §4.3):

```json
[
  {"id": "block:sysA",  "content": "You are a helpful assistant."},
  {"id": "block:u1",    "content": "Hello"},
  {"id": "block:a1",    "content": "Hi! How can I help?"},
  {"id": "block:u2",    "content": "Summarize the above."}
]
```

Independent implementations MUST reproduce these bytes for the given snapshot.

### 12.9 Provider Thread Example with Pre/Post Context (Normative)

Given this example snapshot structure at `@t0` containing pre-context (offset < 0) and post-context (offset > 0):

```json
{
  "spec_version": "PACT/1.0.0",
  "root": {
    "id": "root-2",
    "children": [
      {
        "id": "sys-2",
        "nodeType": "^sys",
        "children": [
          {"id": "block:sysB",  "offset": 0, "content": "System header B"}
        ]
      },
      {
        "id": "seq-2",
        "nodeType": "^seq",
        "children": [
          {
            "id": "seg:10",
            "nodeType": "seg",
            "children": [
              {"id": "block:pre1",     "offset": -1, "content": "Pre-context hint"},
              {"id": "block:core1",    "offset": 0,  "content": "Hello with context"},
              {"id": "block:post1",    "offset": 1,  "content": "status: ok"}
            ]
          }
        ]
      },
      {
        "id": "ah-2",
        "nodeType": "^ah",
        "children": [
          {"id": "block:pre2",   "offset": -1, "content": "AH pre"},
          {"id": "block:core2",  "offset": 0,  "content": "Working..."},
          {"id": "block:post2",  "offset": 1,  "content": "Interim note"}
        ]
      }
    ]
  }
}
```

The provider thread serialization MUST be in this exact order (regions: `^sys` → `^seq` oldest→newest → `^ah`; within each, canonical sibling order per §4.3, i.e., pre < core < post):

```json
[
  {"id": "block:sysB",    "content": "System header B"},
  {"id": "block:pre1",    "content": "Pre-context hint"},
  {"id": "block:core1",   "content": "Hello with context"},
  {"id": "block:post1",   "content": "status: ok"},
  {"id": "block:pre2",    "content": "AH pre"},
  {"id": "block:core2",   "content": "Working..."},
  {"id": "block:post2",   "content": "Interim note"}
]
```

Independent implementations MUST reproduce these bytes for the given snapshot.

## 13. Temporal vs Structural

- Structural invariants (e.g., depth, offset, placement, canonical order) apply identically in the working state and in snapshots.  
- Temporal invariants (e.g., `age`, `born_turn`, TTL/cadence projection rules) are only meaningful in snapshots (turns).  
- The working state has no time‑derived properties; it is free to mutate until commit, subject to structural and validation rules.  

## 14. Selector Validation Rules

- Working State (no `@t`):
  - `{}` and `[]` act as filters only; `age` and `born_turn` are invalid.
  - Edits are permitted: `{}` can assign identity; `[]` can assign lifecycle; comparisons are invalid during edit operations.
- Turns (`@t…`): read-only; edit-intent selectors MUST error.
- Uniqueness:
  - `#key` MUST resolve to ≤1 node; duplicates MUST raise `AmbiguousKey` (or equivalent).
  - `{ id="…" }` uses equality only; if both `key` and `id` are present and mismatch, error `MismatchError`.
- Deep search:
  - `? { … }` is single-step.
  - `?? { … }` is recursive (preorder by default).
  - Chaining `? ?` is invalid.
- After `dN`: entering `seg` is implicit; write `seg{…}` only when filtering/operating on the `seg` node itself.

[← 01-architecture](01-architecture.md) | [↑ Spec Index](../README.md) | [→ 03-lifecycle-ttl](03-lifecycle-ttl.md)