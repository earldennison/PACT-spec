# PACT: Positioned Adaptive Context Tree Specification

**Complete Specification Document**

*This document combines all chapters of the PACT specification into a single comprehensive reference.*

---

## Table of Contents

1. [00 – Overview](#00-–-overview)
2. [01 – Architecture](#01-–-architecture)
3. [02 – Invariants](#02-–-invariants)
4. [03 – Lifecycle and TTL](#03-–-lifecycle-and-ttl)
5. [04 – Context Selectors](#04-–-context-selectors)
6. [05 – Snapshots and Diffs](#05-–-snapshots-and-diffs)
7. [06 – Debugging and Inspection](#06-–-debugging-and-inspection)
8. [07 – Adoption and Compatibility](#07-–-adoption-and-compatibility)
9. [08 – Reference Implementations](#08-–-reference-implementations)
10. [10 – Annex: Spec-wide Conformance Checklist](#10-–-annex-spec-wide-conformance-checklist)
11. [11 – Annex: Glossary (Non‑Normative)](#11-–-annex-glossary-non‑normative)

---

# 00 – Overview

## What is PACT?

**PACT (Positioned Adaptive Context Tree)** is a deterministic substrate for managing context in large language model (LLM) systems.  
Instead of treating context as an unstructured sequence of messages, PACT defines a **tree with invariants** for placement, lifecycle, and rendering.  

This enables developers to build agent frameworks and runtime systems that are **predictable, auditable, and reproducible** at the level of **context state**.

## Motivation

Most current LLM systems handle context using ad-hoc strategies:
- Sliding windows of recent messages,
- Summarization or compaction heuristics,
- Retrieval-augmented memory buffers,
- Flat templates with developer-defined “slots”.

While these techniques work in practice, they lack **structural guarantees**.  
The result is:
- Non-deterministic context assembly across runs,
- Fragile hacks for memory pressure,
- Difficulty debugging or replaying state,
- Vendor-specific idiosyncrasies.

PACT addresses this gap by formalizing the **structure of context itself**, independent of content strategies.

## Core Ideas

1. **Tree structure**  
   The context is organized into a rooted tree with three top-level regions:  
   - `^sys`: System Header (persistent system instructions)  
   - `^seq`: Sealed sequence of past turns (sealed turns’ cores are immutable)  
   - `^ah`: Active Head (current in-progress turn)

2. **Deterministic placement**  
   Nodes are positioned using **relative offsets** and **depth**; canonical placement and sibling order are defined in [02 – Invariants §4](02-invariants.md).

3. **Lifecycle control**  
   Every node has a **TTL** (time-to-live in cycles). Expiry and cascading cleanup happen deterministically at commit boundaries.

4. **Transactional commits**  
   A new snapshot of the tree is produced atomically each cycle.  
   Rendering is side-effect free: the same snapshot always produces identical **serialized context bytes** for the provider.

5. **Pruning (optional)**  
   When enabled, pruning MUST be deterministic and apply **after TTL expiry**, using the canonical order **priority → age → id**.  
   **Budgeting strategies** (token/node/depth limits, heuristics) are **non-normative** and out of scope; see *Annex — Budgeting Guidance*.

6. **Selectors & snapshots**  
   PACT includes a CSS-like query language (`ctx.select`) to traverse both **space** (tree structure) and **time** (snapshots).  
   History is the immutable timeline of snapshots; `^seq` is the chat log region.

## Scope of the Spec

The PACT specification defines:

- **Invariants**: placement rules, ordering, lifecycle, and **pruning order (optional)**.  
- **Interfaces**: context selectors (`ctx.select`, `ctx.diff`) and snapshot semantics.  
- **Schemas**: minimal required metadata for node headers and snapshots.  
- **Provider mapping**: how the tree linearizes into provider-agnostic threads.  
- **Conformance criteria**: behaviors implementations must satisfy.

It does **not** define:
- Specific memory strategies (summarization, retrieval, etc.),  
- Provider-specific protocols (OpenAI, Anthropic, etc.),  
- Product-specific node types (notifications, scaffolds, etc.).  

## Why it matters

PACT’s novelty is that it treats **context structure as a first-class substrate**, not just the content inside.  
This separation enables:

- **Determinism** — the same snapshot always renders to the same serialized provider-thread bytes, regardless of LLM randomness.  
- **Auditability** — easy to diff and replay snapshots, independent of model output.  
- **Composability** — content strategies (summaries, RAG, tools) can plug into a stable substrate.  
- **Interoperability** — provider-agnostic rendering ensures portability across vendors.  
- **Debuggability** — developers can ask “what changed?” at the context layer, regardless of stochastic LLM outputs.

## Next

- [01 – Architecture](01-architecture.md): defines the regions, node types, and container patterns.  
- [02 – Invariants](02-invariants.md): specifies placement, ordering, lifecycle, and commit rules.  


---

# 01 – Architecture

## Purpose
This chapter defines PACT’s structural architecture:
- the regions of the context tree,
- the canonical node types (and how user-assigned types fit),
- placement rules (offsets & depth),
- and the ordering contract that governs traversal and rendering.

## 1. Context Tree

All PACT implementations MUST maintain a single rooted tree of context components.

### 1.1 Root
- **`^root`** — the root container (never pruned or expired).  
  All regions exist beneath it.

### 1.2 Regions (roots)
The root has three required children (regions):

- **`^sys` — System Header**  
  Persistent system-level instructions/background.  
  Nodes here commonly have `ttl=None`.

- **`^seq` — Conversation Sequence**  
  Sequence of **sealed turns** (`mt`).  
  Turns are appended in order; once sealed, their **core** is immutable.

- **`^ah` — Active Head (structural container)**  
  Holds the current in-progress turn that will be sealed into `^seq` at commit.  
  ⚠️ **Note:** Mutability is **not** a property of `^ah` itself. Editability during the cycle is governed
  by the **Active Turn (`at`)** (see 03 – Lifecycle §5.5). Only at commit is `^ah` sealed into `^seq` as a new `mt`.

## 2. Node Types

PACT defines **three canonical node types** for interoperability and selectors:

- **`mt` (MessageTurn)**  
  A sealed conversational turn under `^seq`.  
  Anchors exactly one `mc` (core) and may contain pre/post content blocks.

- **`mc` (MessageContainer)**  
  The **core container** within a turn. Canonical placement is defined in [02 – Invariants §4.2](02-invariants.md).  
  Holds the central message content (e.g., user request or provider reply).

- **`cb` (ContentBlock)**  
  Leaf/content node: text, tool call, tool result, media, document, etc.  
  Provider‑agnostic. SHOULD carry attributes for interoperability:
  - `role`: `"user" | "assistant" | "tool" | "system" | "other"`
  - `kind`: e.g., `"text" | "call" | "result" | "image" | "summary"`

### 2.1 User‑Assigned Types (allowed and encouraged)
Implementations and developers MAY define additional **user‑assigned node types** to express domain semantics.

- **Namespace required** to avoid collisions, e.g.:
  - `cb:summary`, `cb:sql_query`, `cb:image_caption`, `custom:note`
- **Selectors**:
  - Broad match via `.cb`
  - Specific match via `[nodeType="cb:summary"]` (or any user-defined value)
- **Requirement**: The canonical mapping MUST remain available so cross‑implementation queries like `.cb` still work.

> PACT v0.1 has **no overlay primitive**. “Overlay‑like” annotations (summaries, corrections, redactions) are represented as `cb` nodes (e.g., `nodeType="cb:summary"`) typically placed in **post‑context**.

## 3. Placement
### 3.1 Relative Offsets
Canonical placement (offset semantics and `mc@0`) is defined normatively in [02 – Invariants §4.1–4.2](02-invariants.md). This chapter is non‑normative for placement.

### 3.2 Depth (sealed sequence addressing)
Depth addresses sealed turns in `^seq` from newest to oldest:

- `depth = 1` → most recent sealed turn  
- `depth = 2` → the turn before it, etc.  
- `^ah` (active head) is not depth‑addressable; it is addressed by the root alias.

## 4. Ordering (canonical, total)

Canonical sibling ordering is defined normatively in [02 – Invariants §4.3](02-invariants.md). This chapter defers to that section.

## 5. Lifecycle Metadata (required fields)

Every node MUST carry at least:

- `id` — opaque stable identifier (hash or UUID, impl‑defined)
- `nodeType` — canonical or user‑assigned type (e.g., `cb`, `cb:summary`)
- `offset` — integer (relative placement)
- `ttl` — `None | 0 | N` (cycles)
- `priority` — integer (for pruning order)
- `cycle` — introducing commit’s cycle (for this node version)
- `created_at_ns` — monotonic nanosecond timestamp
- `created_at_iso` — ISO8601 timestamp (human‑readable)
- `creation_index` — monotonic per‑cycle tie‑breaker (0,1,2,...)

## 6. Structural Patterns

### 6.1 Turn pattern


mt (MessageTurn)
├─ cb (offset <0) ← pre-context
├─ mc (offset =0) ← core container
│ └─ cb ← core content
└─ cb (offset >0) ← post-context

### 6.2 Root layout
^root
├─ ^sys (System Header)
├─ ^seq (Sealed sequence of turns)
└─ ^ah (Active head: current unsealed turn)
---

## 7. Commit Model

- At each cycle’s end, the Active Head (`^ah`) is **sealed** into a new `mt` under `^seq`.  
- A fresh Active Head is created for the next cycle.  
- A **snapshot** is produced at commit; rendering has no side effects: same snapshot → identical provider‑thread bytes.

- [02 – Invariants](02-invariants.md): placement, ordering, lifecycle, pruning, commit rules.

---

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

---

# 03 – Lifecycle and TTL

> **Terminology Clarifier — `^ah` vs `at`**  
>
> | Term | Nature | Meaning |  
> |------|--------|---------|  
> | **Active Head (`^ah`)** | Structural | The container node for the in-progress turn. Sealed → `mt` at commit. Not inherently mutable. |  
> | **Active Turn (`at`)** | Temporal | The editable scope of the current cycle. Governs mutability across regions until sealing. |
>
> **Terminology Lint (non-normative)**  
> When writing spec text, refer to **`^ah`** for structure and **`at`** for editability.  
> Phrases like “`^ah` is editable/mutable” SHOULD be avoided; prefer  
> “during the Active Turn (`at`), content is editable until sealing.”

## 1. Purpose
This section defines the **temporal semantics** of PACT:
- how cycles are defined,
- how Time-To-Live (TTL) works,
- when expiry occurs,
- how cascading cleanup is applied,
- and how lifecycle integrates with sealing and snapshots.

These rules ensure context evolves deterministically over time.

## 2. Cycles
### 2.1 Definition
A **cycle** is one iteration of context rebuild + commit.  
**One cycle equals one provider interaction (episode). TTL is measured in cycles.**  
Every cycle produces exactly one **snapshot**.

### 2.2 Snapshot Alignment
Each cycle ends with a snapshot commit (`@t0`).  
Snapshots are addressable as:
- `@t0` → current
- `@t-1` → previous
- `@cN` → absolute cycle number

### 2.3 Atomicity
Commits are atomic: a cycle MUST either produce a valid snapshot or fail entirely.  
Snapshots MUST be byte-stable (see [02 – Invariants](02-invariants.md)).

### 2.4 Snapshot Boundary (Normative)
See §7 for the Commit Sequence (Normative). The snapshot boundary is defined by that sequence.  
The snapshot for cycle `N` MUST serialize to the exact provider‑bound bytes for that cycle. Persistence MAY be asynchronous, but logical contents MUST correspond to the post‑commit state used to produce the provider bytes.

## 3. TTL (Time To Live)
### 3.1 TTL Semantics
Each node has a `ttl` field interpreted in cycles:
- `None` → persistent (never expires automatically)
- `0` → expires at the next commit boundary
- `N > 0` → persists for N additional cycles after creation

### 3.2 Evaluation
TTL is checked at **commit**. Expired nodes MUST be removed before pruning.

### 3.3 Decrement vs Check
Implementations MAY store TTL as “remaining cycles” (decrement each commit) or as “expiry cycle” (check current cycle against expiry).  
Both are valid if semantics are preserved.

## 4. Expiry and Removal
### 4.1 Expiry Timing
Nodes with `ttl=0` MUST expire immediately at the next commit.  
Nodes with `ttl=N` MUST expire exactly after N cycles.

### 4.2 Cascade Rules
If expiry empties a container that is marked removable, that container MUST also be removed in the same commit.  
Root (`^root`) and region containers (`^sys`, `^seq`, `^ah`) MUST NOT be removed.

### 4.3 Reference Safety
Nodes with live references MUST NOT be removed (expired or pruned). If liveness cannot be determined, implementations MUST preserve such nodes conservatively and MAY defer expiry across commits to satisfy this requirement.

## 5. Sealing & Snapshots
### 5.1 Active Head
There MUST be at most one `^ah` node per snapshot. At commit, the `^ah` is sealed
into history as a new `mt` under `^seq`. Once sealed, the `mc@0` core is immutable.
`^ah` denotes **structure** only; it does not define what is editable.

### 5.2 Sealing & Snapshots
At commit, the Active Head is sealed into history and a snapshot is emitted. Sealed turns and snapshots are immutable artifacts. Sealing affects mutability; `^ah` is a structural pointer, while mutability is temporal and applies to the Active Turn (`at`).

### 5.3 Effective History
After sealing, implementations MAY add new nodes (e.g., summaries, redactions, corrections) anywhere permitted by placement rules. Implementations MUST NOT mutate sealed nodes; changes MUST be expressed by adding new referencing blocks. PACT does not define any semantic override among such nodes. Rendering remains purely structural and deterministic: nodes are serialized in canonical order; any higher-level interpretation (e.g., "use a summary instead of originals") is an implementation choice outside this specification. Conformance is judged solely on structural serialization; semantic use of additions (summaries, redactions, corrections) is out of scope.

Implementations MUST NOT advertise semantic precedence of summaries/corrections as part of PACT conformance.

### 5.4 Determinism
Given the same prior snapshots and the same set of non‑expired additions, the effective history serializes to the same provider‑input bytes.

### 5.5 Edit Scope
**Active Turn (`at`)** — the *temporal edit scope* of the current cycle.

- Begins at cycle start and ends at commit.  
- All nodes created during the cycle (in `^ah` or other regions) are editable until sealing.  
- After sealing, further modifications MUST be expressed as new nodes, never by mutating sealed bytes.

This clarifies that editability is governed by `at`, while `^ah` is merely the structural container
of the turn that will be sealed.

 

## 6. Attempts & Controls
### 6.1 Stop/Edit/Re‑Run
Within a cycle, provider calls produce attempts recorded in telemetry with statuses: running, completed, canceled, error. Attempts are not part of history. If a user stops an attempt, content within the Active Turn remains editable; the system may modify or replace content and re‑run the provider. Only upon commit is the turn sealed into ^seq and a snapshot emitted. Earlier attempts remain visible to operators for observability and audit in the control plane, but do not affect sealed history or snapshots.

### 6.2 Attempt Record Schema
Attempt records MUST include: `attempt_id`, `cycle_id`, `status`, `started_at`, `ended_at`, `provider`, `model`, `params(hash)`, `token_in`, `token_out`, `error(optional)`.

## 7. Order of Lifecycle Operations
At commit, lifecycle MUST be applied in this sequence:

1. **TTL Expiry**  
   Remove all nodes whose TTL has expired. Apply cascading cleanup.
2. **Pruning/Compaction (if any)**  
   Apply pruning or compaction policies if configured (see [02 – Invariants](02-invariants.md), §7).  
3. **Sealing**  
   Seal the Active Head into `^seq` as a new `mt`.  
   Create a fresh `^ah` for the next cycle.
4. **Snapshot Commit**  
   Produce a stable snapshot (`@t0`).  

This ordering ensures reproducibility across implementations.

> **Commit Sequence (Normative)** — At each cycle's commit: 1. TTL expiry and cascading cleanup; 2. Pruning/compaction (if implemented) in canonical order; 3. Seal `^ah` into a new `mt` under `^seq`; 4. Produce snapshot `@t0`. (Conformance requires identical results for identical inputs.)

## 6.1 Summarization/Rebuild (Non‑Normative)

Implementations MAY rebuild the current turn each cycle by adding new summarization/redaction/correction nodes and retiring heavier originals via TTL/budget. This enables different runtime strategies without changing sealed records or the PACT rendering rules.

## 8. Examples

### 8.1 TTL Expiry

Cycle 10:
├─ cb (ttl=0) ← expires at commit
├─ cb (ttl=2) ← survives until cycle 12
└─ cb (ttl=None) ← persistent



### 8.2 Cascade Cleanup
mt
└─ mc
└─ cb (ttl=0)

### 8.3 Sealing
^ah
├─ mc (offset=0)
│ └─ cb (role="user")
└─ cb (offset=+1, role="tool")
---


---

# 04 – Context Selectors

## 1. Purpose
This section defines the **context selector language** (`ctx.select`) for querying nodes in a PACT tree.  
Selectors allow developers to traverse both **space** (tree structure) and **time** (snapshots).  

Selectors MUST be deterministic and side-effect free.

## 2. Selector Model
### 2.1 Input
- A selector string (CSS-like syntax).  
- An optional snapshot reference (`@t0`, `@t-1`, `@c42`).  

### 2.2 Output
- **Non-range** (single snapshot `@tN` / `@cN` / omitted ⇒ `@t0` / wildcard `@*`):  
  `ctx.select` MUST return an **ordered list of node IDs** in canonical sibling order.
- **Range present** (`@tA..@tB` or `@tA:@tB`):  
  `ctx.select` MUST return a **RangeDiffLatestResult** (pairwise diffs, snapshots sorted DESC: newest→oldest).

**Rationale.** Ranges imply evolution over time; the natural result is “what changed” rather than a flat union of IDs. This preserves stable, pre-existing behavior for non-range calls (including `@*`).

## 3. Syntax Elements
### 3.1 Roots
- `^sys` — System Header  
- `^seq` — Sealed sequence of turns  
- `^ah` — Active Head (in-progress turn)  
- `^root` — Root container

### 3.2 Types
- `.mt` — MessageTurn  
- `.mc` — MessageContainer  
- `.cb` — ContentBlock  
- Namespaced user-assigned types MUST also be supported. A type token like `.cb:summary` MUST match nodes whose `nodeType` equals `"cb:summary"`.
- Equivalence (Normative): `.cb:summary` and `[nodeType='cb:summary']` MUST be treated as equivalent forms and MUST return identical results.
- `kind` is not a type; query it via attributes (e.g., `.cb[kind='tool_call']`). Bare `.tool_call` is not standardized and MUST NOT be required for conformance.

### 3.3 IDs
- `#<id>` — select by node ID (opaque string)

### 3.4 Pseudo-classes
- `:pre` — nodes with `offset < 0`  
- `:core` — nodes with `offset = 0`  
- `:post` — nodes with `offset > 0`  
- `:depth(n)` — select a turn at depth `n` in `^seq` (newest = 1)  
- `:depth(n1,n2,...)` — select turns at multiple depths (comma-separated list)  
- `:depth(n1-n2)` — select turns within inclusive depth range `n1` through `n2` (newest = 1)  
- `:first` — first sibling among a set  
- `:last` — last sibling among a set  
- `:nth(n)` — nth sibling (1-based index)

Offset-based pseudos map to canonical offset semantics defined in [02 – Invariants §4.1–4.2](02-invariants.md).

### 3.5 Attributes
Selectors MUST support attributes:  
`[offset] [ttl] [priority] [cycle] [created_at_ns] [created_at_iso] [nodeType] [id] [role] [kind]`

Comparison operators: `=`, `!=`, `<`, `<=`, `>`, `>=`.
 
Implementations create attributes via node headers defined in [02 – Invariants §3] (see §3.5 “Attribute Creation and Stability”). Missing optional attributes compare as `null` per §5.5; mandatory headers MUST be populated.

### 3.6 Combinators
- Descendant: `A B`  
- Child: `A > B`

### 3.7 Snapshots
Snapshots are addressable with `@t0` (current), `@t-1` (previous), `@cN` (absolute cycle), or `@*` (all).
If omitted, the snapshot reference MUST default to `@t0`. Implementations MUST NOT require explicit `@t0`.

Range addressing across snapshots MUST be supported with **inclusive** semantics:
- `@t-<start>..<end>` — inclusive range using double-dots  
- `@t-<start>:<end>` — inclusive range using a colon (**interchangeable** with `..`)

**Contract**
- Presence of a SnapshotRange ⇒ return **RangeDiffLatestResult**.  
- Otherwise ⇒ return **ordered list of IDs**.

**Examples**
```text
# Non-range (IDs)
ctx.select("@t0 ^seq .mt:depth(1) .mc > .cb[role='assistant']")

# Range (RangeDiffLatestResult)
ctx.select("@t-3..@t0 ^seq .mt .cb[nodeType='cb:summary']")
     "query": "@t-3..@t0 ^seq .mt .cb[nodeType='cb:summary']",
     "snapshots": [ { "label":"@t0" }, { "label":"@t-1" }, { "label":"@t-2" }, { "label":"@t-3" } ],
     "diffs": [
       { "from":{"label":"@t0"}, "to":{"label":"@t-1"}, "added_ids":[], "removed_ids":["cb:sum:c101"], "changed":[] },
       { "from":{"label":"@t-1"}, "to":{"label":"@t-2"}, "added_ids":["cb:sum:c102"], "removed_ids":[], "changed":[] },
       { "from":{"label":"@t-2"}, "to":{"label":"@t-3"}, "added_ids":[], "removed_ids":[], "changed":[] }
     ],
     "mode": "pairwise"
   }
```

## 4. Grammar (EBNF)
```ebnf
# SnapshotRange presence affects ONLY the return type (IDs vs RangeDiffLatestResult).
Selector   ::= [Snapshot] Group { "," Group }
Group      ::= Chain
Chain      ::= Step { Combinator Step }
Step       ::= Simple
Combinator ::= " " | ">"
Simple     ::= [Root] [ID] [Type] [Attr*] [Pseudo*] | "*"

Snapshot        ::= SnapshotAtom | SnapshotRange
SnapshotAtom    ::= "@" ( "t" [ "-" ] Digit+ | "c" Digit+ | "*" )
SnapshotRange   ::= SnapshotAtom ".." SnapshotAtom   # inclusive; ":" interchangeable with ".."
Root            ::= "^" ("sys" | "seq" | "ah" | "root")
ID              ::= "#" Identifier
Type            ::= "." Identifier
Attr            ::= "[" Key [Op Value] "]"
Pseudo          ::= ":" Identifier [ "(" ValueList ")" ]

ValueList  ::= Value { "," Value }
Value      ::= Number | String | Identifier | Range
Range      ::= Number "-" Number
Number     ::= [ "-" ] Digit+ [ "." Digit+ ]
String     ::= '"' StringChar* '"' | "'" StringChar* "'"
Identifier ::= Letter ( Letter | Digit | "_" | "-" | ":" )*
Key        ::= Identifier
Op         ::= "=" | "!=" | "<" | "<=" | ">" | ">="
Letter     ::= "a".."z" | "A".."Z"
Digit      ::= "0".."9"
StringChar ::= [^"'] | "\\" ( '"' | "'" | "\\" )
```

Constraints:
- Endpoints MUST be the same kind: "@t..@t" or "@c..@c".
- "@*" MUST NOT appear inside a range.
- Range is inclusive of both endpoints.

### Range Select Output — RangeDiffLatestResult (Normative)

When a SnapshotRange is present, `ctx.select` returns:

RangeDiffLatestResult {
  "query": string,                      // original selector
  "snapshots": SnapshotRef[],           // expanded & resolved, sorted DESC by cycle (newest→oldest)
  "diffs": DiffBlock[],                 // pairwise steps: (s0→s1), (s1→s2), ...
  "mode": "pairwise",                   // reserved for future (e.g., "nodewise","baseline")
  "limits"?: { "maxSnapshots"?:number, "maxChangesPerSnapshot"?:number, "truncated"?:boolean },
  "warnings"?: string[],
  "errors"?: Array<{ "code":string, "message":string, "fatal":boolean }>
}

SnapshotRef { "kind":"t"|"c", "value":number, "label":string, "cycle":number }

DiffBlock {
  "from": SnapshotRef,                  // newer snapshot
  "to":   SnapshotRef,                  // older snapshot
  "added_ids":   string[],              // present in 'from' but not in 'to'
  "removed_ids": string[],              // present in 'to'   but not in 'from'
  "changed": Array<{
    "id": string,
    "fields": string[],                 // headers that differ (e.g., ["ttl","content_hash","parent","offset"])
    "delta"?: Record<string,{ "from":any, "to":any }>,
    "content_diff"?: {                  // present iff options.includeContent=true AND "content_hash" changed
      "mode":"unified"|"jsonpatch"|"tokens",
      "mime"?: string,
      "summary": { "bytesNewer":number, "bytesOlder":number, "identical":boolean, "truncated":boolean },
      "diff"?: string,
      "patch"?: any,
      "hunks"?: any[]
    }
  }>,
  "stats"?: { "added":number, "removed":number, "changed":number }
}

Changed detection (headers)
- A node is “changed” iff the same id is present in both snapshots of the pair and any tracked header differs
  (e.g., ttl, priority, parent, offset, nodeType, role, kind, content_hash, created_at_ns, creation_index).
  delta carries scalar old/new values. If content_hash changed and options.includeContent=true,
  implementations SHOULD attach a content_diff.

Ordering and determinism
- snapshots sorted newest→oldest (DESC by cycle).
- Within each DiffBlock, added_ids/removed_ids SHOULD be ordered by the canonical sibling order of the from snapshot,
  then by id.
- Deterministic: identical inputs produce identical outputs byte-for-byte.

### ctx.select Options (applies when a SnapshotRange is present)

Signature:
  ctx.select(selector, options?)

Options:
- includeContent?: boolean
  When true, and a node’s content_hash is in changed.fields, attach a content_diff to that changed item.

- contentMode?: "unified"|"jsonpatch"|"tokens"
  Default "unified". Controls the content_diff representation.

- contextLines?: number
  Unified diff context lines (default 3).

- materialize?: ("content"|"mime"|"encoding"|"content_hash")[]
  When present, servers MAY inline minimal fields needed to compute or validate diffs.

- maxSnapshots?: number
  Cap range expansion.

- maxChangesPerSnapshot?: number
  Cap size of each DiffBlock; set limits.truncated=true on truncation.

Non-range calls ignore these options and continue to return IDs.

## 5. Attribute Comparison Rules

### 5.1 Data Type Detection
Attribute values MUST be compared using appropriate data types:

- **Numeric attributes**: `offset`, `ttl`, `priority`, `cycle`, `created_at_ns`
- **String attributes**: `nodeType`, `id`, `role`, `kind`, `created_at_iso`

### 5.2 Numeric Comparison
For numeric attributes, implementations MUST:
1. Parse attribute values as numbers (integers or floats)
2. Apply numeric comparison semantics for `<`, `<=`, `>`, `>=`
3. Treat `None` or `null` values as distinct from numeric values
4. Use exact equality for `=` and `!=`

```
[ttl>5]      # numeric: ttl greater than 5
[offset<0]   # numeric: offset less than zero  
[cycle=42]   # numeric: exact equality
```

### 5.3 String Comparison  
For string attributes, implementations MUST:
1. Use case-sensitive string comparison by default
2. Apply lexicographic ordering for `<`, `<=`, `>`, `>=`
3. Support exact string matching for `=` and `!=`
4. Treat empty strings as distinct from null/undefined

```
[role='user']           # exact string match (case-sensitive)
[nodeType='cb:summary'] # exact match with namespace
[id!='abc123']          # string inequality
```

### 5.4 Mixed Type Handling
When attribute type is ambiguous:
1. Attempt numeric parsing first for comparison operators `<`, `<=`, `>`, `>=`
2. Fall back to string comparison if numeric parsing fails
3. For `=` and `!=`, use type-preserving comparison
4. Implementations SHOULD document their type coercion rules

### 5.5 Special Values
- **`None`/`null`**: Only matches with `=` or `!=`, never with ordering operators
- **Empty string `""`**: Treated as valid string, distinct from `None`
- **Boolean**: Convert to string ("true"/"false") for comparison

## 6. Examples

# all pre-context content blocks in active head

ctx.select("@t0 ^ah :pre .cb")

# assistant replies from last sealed turn
ctx.select("^seq .mt:depth(1) .mc > .cb[role='assistant']")

# expired or soon-to-expire blocks
ctx.select("^root .cb[ttl<=1]")

# a specific node by ID
ctx.select("#abc123def")

# last three sealed turns, all user messages
ctx.select("^seq .mt:depth(1,2,3) .cb[role='user']")

# last three sealed turns (range form), all user messages
ctx.select("^seq .mt:depth(1-3) .cb[role='user']")

# last five sealed turns (range form)
ctx.select("^seq .mt:depth(1-5)")

# select across snapshot range t-5 through t-1 (inclusive)
ctx.select("@t-5..-1 ^ah .cb")

# find a specific node across all available snapshots
ctx.select("@* #abc123def")

# position node N under a specific sealed turn (depth targeting)
# select the target turn at depth 3 and then place N as post-context via offset>0
# (actual placement is done by the implementation API; selector here shows targeting)
ctx.select("^seq .mt:depth(3)")

## 6.2 Canonical Selector Tests (Normative)

#### Range (returns RangeDiffLatestResult)
- MUST return `RangeDiffLatestResult` for any selector containing a SnapshotRange.
- Snapshots MUST be sorted newest→oldest (DESC by cycle) in the `snapshots` array.
- `diffs` MUST be pairwise: (s0→s1), (s1→s2), …

#### Non-range (returns IDs)
- MUST return ordered array of node IDs for single-snapshot or `@*` queries.

Given this minimal snapshot fixture at `@t0`:

```json
{
  "root": {
    "children": [
      { "id": "sys-1", "nodeType": "^sys", "children": [
        { "id": "cb:sysA", "nodeType": "cb", "role": "system", "kind": "text", "offset": 0, "content": "S" }
      ]},
      { "id": "seq-1", "nodeType": "^seq", "children": [
        { "id": "mt:1", "nodeType": "mt", "children": [
          { "id": "cb:u1", "nodeType": "cb", "role": "user", "kind": "text", "offset": 0, "ttl": 2, "content": "U1" }
        ]},
        { "id": "mt:2", "nodeType": "mt", "children": [
          { "id": "cb:a1", "nodeType": "cb", "role": "assistant", "kind": "text", "offset": 0, "ttl": 1, "content": "A1" }
        ]}
      ]},
      { "id": "ah-1", "nodeType": "^ah", "children": [
        { "id": "cb:u2", "nodeType": "cb", "role": "user", "kind": "text", "offset": 0, "content": "U2" }
      ]}
    ]
  }
}
```

Implementations MUST produce exactly these results (IDs and order):

1. `@t0 ^sys .cb` → `["cb:sysA"]`
2. `@t0 ^seq .mt:depth(1)` → `["mt:2"]`
3. `@t0 ^seq .mt:depth(1,2)` → `["mt:1","mt:2"]` (older→newer)
4. `@t0 ^seq .mt:depth(1-2) .mc > .cb` → `["cb:u1","cb:a1"]`
5. `@t0 #cb:u2` → `["cb:u2"]`

## 6.3 Additional Range Fixture (Normative)

To validate range depth selectors, the following minimal snapshot fixture at `@t0` MUST produce the listed results:

```json
{
  "root": {
    "children": [
      { "id": "seq-2", "nodeType": "^seq", "children": [
        { "id": "mt:1", "nodeType": "mt", "children": [ { "id": "cb:u1", "nodeType": "cb", "role": "user", "kind": "text", "offset": 0, "content": "U1" } ] },
        { "id": "mt:2", "nodeType": "mt", "children": [ { "id": "cb:u2", "nodeType": "cb", "role": "user", "kind": "text", "offset": 0, "content": "U2" } ] },
        { "id": "mt:3", "nodeType": "mt", "children": [ { "id": "cb:u3", "nodeType": "cb", "role": "user", "kind": "text", "offset": 0, "content": "U3" } ] }
      ]}
    ]
  }
}
```

- Query: `@t0 ^seq .mt:depth(1-3) .cb[role='user']`
  Expect: `["cb:u1","cb:u2","cb:u3"]` (older→newer across matched turns)


## 7. Conformance Checklist

An implementation is conformant if:

1. Roots ^sys, ^seq, ^ah, ^root MUST be supported.
2. Types .mt, .mc, .cb MUST be supported.
3. User-assigned nodeType MUST be matchable via namespaced type tokens (e.g., `.cb:summary`) and via attributes (e.g., `[nodeType='cb:summary']`).
4. IDs MUST be matched exactly (case-sensitive).
5. All specified pseudos MUST be implemented.
6. Attributes MUST support all comparison operators.
7. Combinators (descendant, child) MUST be supported.
8. Snapshots (@t0, @t-1, @cN, @*) MUST be supported.
9. Results MUST be ordered canonically.
10. Queries MUST be pure (no side effects).
11. Invalid selectors MUST raise errors.
12. `:depth(n1,n2,...)` comma-separated lists MUST be supported.
13. Numeric vs string attribute comparison rules MUST be followed.
14. Data type detection for attributes MUST be implemented correctly.
15. Default snapshot if omitted MUST be `@t0` (see §3.7); implementations MUST NOT require explicit `@t0` for current snapshot queries.
16. Snapshot range addressing (`@tA..B` and `@tA:B`) MUST be supported with inclusive semantics and both operators MUST be interchangeable.

- MUST parse SnapshotRange and reject mixed kinds ("@t..@c") or wildcard endpoints ("@*" in ranges).
- MUST return RangeDiffLatestResult when a SnapshotRange is present in the selector.
- MUST return a list of IDs when the selector targets a single snapshot or "@*".
- MUST sort snapshots newest→oldest and compute pairwise diffs in that order.
- MUST be deterministic (stable ordering and byte-identical outputs for identical inputs).
- SHOULD expose limits and set truncated=true when caps are hit.

### 7.1 Golden Tests (Normative)

Given the minimal snapshot fixture in §6.2, implementations MUST produce exactly these ordered ID results:

- Query: `@t0 ^sys .cb`
  Expect: `["cb:sysA"]`

- Query: `@t0 ^seq .mt:depth(1)`
  Expect: `["mt:2"]`

- Query: `@t0 ^seq .mt:depth(1,2)`
  Expect: `["mt:1","mt:2"]` (older→newer)

- Query: `@t0 ^seq .mt:depth(1-2) .mc > .cb`
  Expect: `["cb:u1","cb:a1"]`

- Query: `@t0 ^seq .mt:depth(1) > .cb`
  Expect: `["cb:a1"]`

- Query: `@t0 #cb:u2`
  Expect: `["cb:u2"]`

- Query: `@t0 .cb[role='assistant']`
  Expect: `["cb:a1"]`

- Query: `@t0 ^seq .mt:depth(1-2) .cb[ttl<=1]`
  Expect: `["cb:a1"]`

- Query: `@t0 ^seq .mt:depth()`
  Expect: MUST raise an error (invalid selector: empty depth)

- Query: `@t0 ^seq .mt:depth(3) .cb[role='user']`
  Expect: `[]` (empty match, not an error)



---

# 05 – Snapshots and Diffs

## 1. Purpose
This section defines how **snapshots** are produced, addressed, and compared.  
Snapshots make context state reproducible, diff-able, and auditable.

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

Range addressing across snapshots MUST be supported with inclusive semantics:

- `@t-<start>..<end>` — inclusive range using double-dots
- `@t-<start>:<end>` — inclusive range using a colon (interchangeable with `..`)

Notes:

- Negative indices are relative to the current snapshot: `@t-1` is the immediate previous snapshot, `@t-2` two cycles back, etc.
- `..` and `:` MUST be treated as interchangeable range operators and produce identical results.

## 3. Export and Import
### 3.1 Export
Implementations MUST support exporting snapshots in serialized form including:
- All nodes and required metadata,  
- Canonical sibling ordering (per [02 – Invariants §4.3](02-invariants.md)),  
- Stable IDs.
 
 

### 3.2 Import
Implementations SHOULD support re-importing snapshots for replay.  
Replay MUST yield the same serialization as the original.

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

### 5.3 Snapshot Range Addressing

Selectors MAY reference multiple snapshots using ranges. For example:

```
ctx.select("@t-5..-1 ^ah .cb")
```

This MUST select across snapshots from `t-5` through `t-1` inclusive. The following is equivalent and MUST return identical results:

```
ctx.select("@t-5:-1 ^ah .cb")
```

## 6. Conformance Checklist

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

[← 04-selectors](04-selectors.md) | [↑ Spec Index](../README.md) | [→ 06-debugging](06-debugging.md)

---

# 06 – Debugging and Inspection

## 1. Purpose
This section defines how PACT implementations MUST support **observability**:  
- what metadata nodes carry for inspection,  
- how selectors and diffs are exposed for debugging,  
- and how determinism guarantees reproducible audits.

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

# last two sealed turns, assistant responses only
ctx.select("^seq .mt:depth(1,2) .cb[role='assistant']")

# expired or soon-to-expire nodes
ctx.select("^root .cb[ttl<=1]")
```

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

### 5. Snapshot Replay
### 5.1 Export

Implementations MUST support exporting snapshots in a serialized form that includes:

All nodes with required metadata,

Canonical ordering preserved (per [02 – Invariants §4.3](02-invariants.md)),

Stable IDs.

### 5.2 Import

Implementations SHOULD support re-importing a snapshot for replay.
Replay MUST yield the same serialization output as the original.

## 6. Auditing Guarantees

Snapshots are byte-stable: given the same input, serialization is identical.

Selectors are pure: repeated queries yield the same results.

Diffs are complete: all changes are attributable to TTL, pruning, or mutation during the Active Turn.

Debugging is structural: the same invariants apply regardless of LLM stochasticity.

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


---

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
3. **Optional pruning/compaction** (non-normative budgeting; pruning is optional for conformance).  
4. **Selectors & diffs** (debugging, inspection).  

Each layer adds determinism and observability without invalidating earlier layers.

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

- Placement guidance:
- Attach RAG `cb` under the relevant turn’s post‑context (`offset > 0`) or into `^ah` during the current cycle.
- Use `ttl` to expire retrievals deterministically; pruning SHALL follow priority → age → id.

## 5. Versioning
### 5.1 Spec Version
Snapshots SHOULD include a `spec_version` field (e.g., `"PACT/0.1"`).  
This ensures replay across different runtimes is compatible.

### 5.2 Forward Compatibility
- Older implementations MUST ignore unknown attributes or node types.  
- Extensions MUST not alter the meaning of canonical fields.  

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

# 08 – Reference Implementations

## 1. Purpose
This appendix provides **reference implementations** for critical PACT algorithms to ensure consistent behavior across implementations. These are normative examples that implementations SHOULD follow for interoperability.

## 2. Content Hash Calculation

### 2.1 Algorithm
Content hashing MUST be deterministic and collision-resistant. Implementations MUST use the following algorithm for interoperability:

```python
import hashlib
import json

def calculate_content_hash(node):
    """
    Calculate a stable content hash for a PACT node.
    
    Args:
        node: Node object with content and metadata
        
    Returns:
        str: Hexadecimal hash string (SHA-256)
    """
    # Extract only content-relevant fields, excluding metadata
    hashable_content = {
        'content': node.get('content', ''),
        'kind': node.get('kind', ''),
        'role': node.get('role', ''),
        # Include any custom content attributes
        **{k: v for k, v in node.items() 
           if k.startswith('content_') or k.startswith('data_')}
    }
    
    # Canonical JSON serialization (sorted keys, no whitespace)
    canonical_json = json.dumps(hashable_content, 
                               sort_keys=True, 
                               separators=(',', ':'),
                               ensure_ascii=True)
    
    # SHA-256 hash
    return hashlib.sha256(canonical_json.encode('utf-8')).hexdigest()
```

### 2.2 Excluded Fields
Content hash MUST NOT include:
- Lifecycle metadata: `ttl`, `priority`, `cycle`, `created_at_ns`, `created_at_iso`
- Tree structure: `id`, `offset`, `parent_id`, `children`
- Implementation-specific: internal pointers, cache values, etc.

### 2.3 Hash Stability
Content hash MUST remain stable across:
- Node movement within tree
- TTL changes
- Priority adjustments
- Timestamp updates

## 3. Concurrent Access Patterns

### 4.1 Threading Model
PACT implementations SHOULD assume single-threaded access during commits, but MAY support concurrent readers:

```python
import threading
from typing import Optional

class PACTTree:
    def __init__(self):
        self._snapshot_lock = threading.RLock()
        self._current_snapshot: Optional[Dict] = None
        self._read_count = 0
        
    def get_snapshot(self, address: str = "@t0"):
        """Thread-safe snapshot access for selectors/diffs."""
        with self._snapshot_lock:
            # Multiple concurrent readers allowed
            self._read_count += 1
            try:
                return self._get_snapshot_impl(address)
            finally:
                self._read_count -= 1
    
    def commit_cycle(self, mutations):
        """Exclusive write access during commit."""
        with self._snapshot_lock:
            # Wait for active readers to complete
            while self._read_count > 0:
                time.sleep(0.001)  # Brief yield
            
            # Atomic commit
            new_snapshot = self._apply_mutations(mutations)
            self._current_snapshot = new_snapshot
            return new_snapshot
```

### 4.2 Memory Safety
Node references and TTL management:

```python
from weakref import WeakSet

class NodeReference:
    """Safe reference to PACT nodes with TTL interaction."""
    
    def __init__(self, node_id: str, tree: PACTTree):
        self.node_id = node_id
        self._tree_ref = weakref.ref(tree)
        
        # Register with tree to prevent premature TTL expiry
        tree._register_reference(self)
    
    def get_node(self):
        """Get current node, returns None if expired/pruned."""
        tree = self._tree_ref()
        if tree is None:
            return None
        return tree.get_node_by_id(self.node_id)
    
    def __del__(self):
        """Cleanup reference on destruction."""
        tree = self._tree_ref()
        if tree is not None:
            tree._unregister_reference(self)
```

## 5. Canonical Ordering Implementation

### 5.1 Comparison Function
Deterministic sibling ordering implementation:

```python
from functools import cmp_to_key
from typing import List, Dict, Any

def compare_siblings(node_a: Dict[str, Any], node_b: Dict[str, Any]) -> int:
    """
    Compare two sibling nodes according to PACT canonical order.
    
    Returns:
        -1 if node_a < node_b
         0 if node_a == node_b  
         1 if node_a > node_b
    """
    # 1. Compare offset (ascending)
    offset_a = node_a.get('offset', 0)
    offset_b = node_b.get('offset', 0)
    if offset_a != offset_b:
        return -1 if offset_a < offset_b else 1
    
    # 2. Compare created_at_ns (ascending)
    time_a = node_a.get('created_at_ns', 0)
    time_b = node_b.get('created_at_ns', 0)
    if time_a != time_b:
        return -1 if time_a < time_b else 1
    
    # 3. Compare creation_index (ascending)
    index_a = node_a.get('creation_index', 0)
    index_b = node_b.get('creation_index', 0)
    if index_a != index_b:
        return -1 if index_a < index_b else 1
    
    # 4. Compare id (lexicographic ascending)
    id_a = node_a.get('id', '')
    id_b = node_b.get('id', '')
    if id_a < id_b:
        return -1
    elif id_a > id_b:
        return 1
    else:
        return 0

def sort_siblings(siblings: List[Dict[str, Any]]) -> List[Dict[str, Any]]:
    """Sort siblings according to PACT canonical order."""
    return sorted(siblings, key=cmp_to_key(compare_siblings))
```

### 5.2 Tree Traversal
Canonical depth-first traversal:

```python
def traverse_canonical(node: Dict[str, Any], visit_fn):
    """
    Traverse PACT tree in canonical order.
    
    Args:
        node: Current node
        visit_fn: Function called for each node
    """
    # Visit current node
    visit_fn(node)
    
    # Visit children in canonical order
    children = node.get('children', [])
    for child in sort_siblings(children):
        traverse_canonical(child, visit_fn)
```

## 6. Serialization Reference

### 6.1 Provider-Agnostic Serialization
Byte-stable serialization for snapshots:

```python
import json
from typing import Dict, Any

def serialize_snapshot(tree: Dict[str, Any]) -> bytes:
    """
    Serialize PACT tree to byte-stable format.
    
    Args:
        tree: PACT tree root
        
    Returns:
        bytes: Canonical serialization
    """
    def normalize_node(node: Dict[str, Any]) -> Dict[str, Any]:
        """Normalize node for serialization."""
        normalized = {}
        
        # Required fields in fixed order
        for field in ['id', 'nodeType', 'offset', 'ttl', 'priority', 
                     'cycle', 'created_at_ns', 'created_at_iso']:
            if field in node:
                normalized[field] = node[field]
        
        # Content fields
        for field in ['content', 'role', 'kind']:
            if field in node:
                normalized[field] = node[field]
                
        # Children in canonical order
        if 'children' in node:
            normalized['children'] = [
                normalize_node(child) 
                for child in sort_siblings(node['children'])
            ]
        
        return normalized
    
    normalized_tree = normalize_node(tree)
    canonical_json = json.dumps(normalized_tree, 
                               sort_keys=True,
                               separators=(',', ':'),
                               ensure_ascii=True)
    
    return canonical_json.encode('utf-8')
```

## 7. Conformance Testing

### 7.1 Reference Test Cases
Implementations SHOULD pass these reference tests:

```python
def test_content_hash_stability():
    """Test that content hash is stable across metadata changes."""
    node1 = {
        'id': 'test1',
        'content': 'Hello world',
        'role': 'user',
        'ttl': 5,
        'created_at_ns': 1000000
    }
    
    node2 = {
        'id': 'test2',  # Different ID
        'content': 'Hello world',
        'role': 'user', 
        'ttl': 10,      # Different TTL
        'created_at_ns': 2000000  # Different timestamp
    }
    
    # Content hashes should be identical
    assert calculate_content_hash(node1) == calculate_content_hash(node2)

def test_canonical_ordering():
    """Test deterministic sibling ordering."""
    siblings = [
        {'id': 'c', 'offset': 0, 'created_at_ns': 3000},
        {'id': 'a', 'offset': -1, 'created_at_ns': 1000}, 
        {'id': 'b', 'offset': 0, 'created_at_ns': 2000},
        {'id': 'd', 'offset': 1, 'created_at_ns': 4000}
    ]
    
    sorted_siblings = sort_siblings(siblings)
    expected_order = ['a', 'b', 'c', 'd']  # offset, then time, then id
    actual_order = [node['id'] for node in sorted_siblings]
    
    assert actual_order == expected_order

def test_serialization_stability():
    """Test byte-stable serialization."""
    tree = create_test_tree()
    
    # Multiple serializations should be identical
    bytes1 = serialize_snapshot(tree)
    bytes2 = serialize_snapshot(tree)
    
    assert bytes1 == bytes2
```

## 8. Implementation Guidelines

### 8.1 Performance Considerations
- **Token counting:** Cache results where content unchanged
- **Content hashing:** Compute lazily, cache aggressively  
- **Canonical ordering:** Pre-sort during tree construction
- **Serialization:** Stream large trees to avoid memory pressure

### 8.2 Error Handling
Implementations MUST handle these error conditions consistently:
- **Invalid content:** MUST return empty string hash, SHOULD log warning
- **Missing metadata:** MUST use defaults (priority=0, ttl=None, created_at_ns=0)
- **Circular references:** MUST detect and reject during tree construction
- **Hash collisions:** MUST log if detected (extremely unlikely with SHA-256)

[← 07-adoption](07-adoption.md) | [↑ Spec Index](../README.md)

---

# 10 – Annex: Spec-wide Conformance Checklist

This annex aggregates all conformance requirements into one actionable checklist. A new runtime SHOULD be able to implement PACT without reading prose by following the items below end-to-end.

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

## B. Selectors (from 04 – Context Selectors)

MUST (mark one):
- [ ] Yes  [ ] No — Are roots `^sys`, `^seq`, `^ah`, `^root` supported?
- [ ] Yes  [ ] No — Are types `.mt`, `.mc`, `.cb` supported, and do `.cb:summary` and `[nodeType='cb:summary']` return identical results?
- [ ] Yes  [ ] No — Do ID selectors `#<id>` match exactly (case‑sensitive)?
- [ ] Yes  [ ] No — Are pseudos `:pre`, `:core`, `:post`, `:depth(n)`, `:first`, `:last`, `:nth(n)` implemented, including `:depth(n1,n2,...)` lists and `:depth(n1-n2)` ranges?
- [ ] Yes  [ ] No — Are attributes `[offset] [ttl] [priority] [cycle] [created_at_ns] [created_at_iso] [nodeType] [id] [role] [kind]` supported with `= != < <= > >=` and correct numeric/string semantics?
- [ ] Yes  [ ] No — Are combinators descendant (`A B`) and child (`A > B`) supported?
- [ ] Yes  [ ] No — Are `@t0`, `@t-1`, `@cN`, `@*` supported, with default `@t0` if omitted and no requirement for explicit `@t0`?
- [ ] Yes  [ ] No — Are snapshot ranges `@tA..B` and `@tA:B` supported with inclusive semantics, and do both operators (`..`, `:`) produce identical results?
- [ ] Yes  [ ] No — Are results ordered canonically and queries side‑effect free?
- [ ] Yes  [ ] No — Do invalid selectors raise errors?

Reference: 04 – Context Selectors §3–§7.

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

## D. Debugging & Inspection (from 07 – Debugging and Inspection)

MUST (mark one):
- [ ] Yes  [ ] No — Do nodes expose mandatory metadata fields (see A.2)?
- [ ] Yes  [ ] No — Are selectors pure and do they support all required features (see B)?
- [ ] Yes  [ ] No — Does `ctx.diff` report `added`/`removed`/`changed` with `{id, fields[]}` and preserve newer‑snapshot order?
- [ ] Yes  [ ] No — Is there no serialization path that mutates the tree? (see A.8)

SHOULD (mark one):
- [ ] Yes  [ ] No — Does snapshot export preserve canonical ordering and replay/import yield identical serialization?

Reference: 07 – Debugging and Inspection §2–§7.

## E. Commit Sequence (Normative Reference)

At each cycle's commit: 1. TTL expiry and cascading cleanup; 2. Pruning/compaction (if implemented) in canonical order; 3. Seal `^ah` into a new `mt` under `^seq`; 4. Produce snapshot `@t0`.



### Annex: Migration Flag (Non-normative)

Runtime configuration MAY include:
- select_range_returns_diff = true  (default)

If a deployment needs legacy behavior, setting false causes range selects to return a flat, ordered list of IDs as in prior versions; `ctx.rangeDiffLatest(selector, opts?)` SHOULD be provided as a separate API for diffs during the migration period.

---

# 11 – Annex: Glossary (Non‑Normative)

Concise definitions of key PACT terms, with section references.

- Cycle: One rebuild+commit iteration. Exactly one snapshot per cycle. See 03 – Lifecycle §2.
- Snapshot: Byte-stable tree after TTL and sealing; pruning/compaction is optional. `@t0` MUST re‑serialize to provider bytes. See 05 – Snapshots §2.1.
- Sealed (turn/core): A turn (`mt`) sealed from `^ah` into `^seq`. Its `mc@0` core is immutable. See 02 – Invariants §6.2, §6.4.
**Active Head (`^ah`)** — Structural container for the in-progress turn. Exists exactly once per snapshot.
Sealed into `^seq` at commit. Not inherently mutable.
**Active Turn (`at`)** — Temporal scope of editability within the current cycle.
Includes the `^ah` and any other nodes created during this cycle. Ends when commit seals the turn.
- Turn (`mt`): Sealed conversational turn under `^seq`. Anchors exactly one `mc` (core) and may include pre/post `cb` children. See 01 – Architecture §2.
- Core (`mc`): MessageContainer at `offset=0` inside a turn; the main message body. Exactly one per `mt`. See 02 – Invariants §4.2.
- Content Block (`cb`): Leaf/content node (text/call/result/media, etc.) with optional `role` and `kind`. See 01 – Architecture §2.
- Combinator: Selector relationship operators. Descendant (`A B`) matches any descendant; child (`A > B`) matches direct children only. See 04 – Selectors §3.3, §4.
- Effective History: Rendered history computed from sealed turns plus any additions
  (e.g., summaries, redactions, corrections) created after sealing. **PACT does not
  define any semantic precedence or override among such additions**; rendering is
  purely structural and deterministic. Higher-level interpretation (e.g., “use a
  summary instead of originals”) is an implementation decision outside PACT
  conformance. See 03 – Lifecycle §5.3.
- Depth: Addressing sealed turns in `^seq` newest→oldest. `depth=1` is most recent. See 01 – Architecture §3.2.
- Offset: Relative placement within a turn: `<0` pre, `0` core (`mc`), `>0` post. See 02 – Invariants §4.1.
- Regions: Root children `^sys`, `^seq`, `^ah` under `^root`. See 02 – Invariants §2.
- Commit Sequence: TTL → Pruning/Compaction (if implemented) → Seal → Snapshot. Canonical rule in 03 – Lifecycle §7.
- Removable container: A container explicitly marked `removable=true` at creation or by deterministic policy; when all children expire or are pruned, it MUST be removed in the same commit. Root (`^root`) and region containers (`^sys`, `^seq`, `^ah`) MUST NOT be removable. See 02 – Invariants §5.3.1.
- Reference safety: Nodes referenced by live pointers MUST NOT be expired or pruned. When reference liveness is unknown, preserve conservatively. See 03 – Lifecycle §4.3 and 02 – Invariants §5.4.
- Canonical pruning order: If pruning is implemented, nodes MUST be pruned by `priority`, then age (`created_at_ns` ascending), then `id` (lexicographic). See 02 – Invariants §7.1.
- Canonical Order: Sibling ordering is deterministic: `offset` asc → `created_at_ns` asc → `creation_index` asc → `id` lexicographic. See 02 – Invariants §4.3.

- Golden Tests: Normative selector fixture results (04 – Selectors §6.2–§7.1). Implementations MUST produce exactly the listed IDs and order for each query; used for conformance.




---
