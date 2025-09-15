[← 00-overview](00-overview.md) | [↑ Spec Index](../README.md) | [→ 02-invariants](02-invariants.md)

# 01 – Architecture

## Purpose
This chapter defines PACT’s structural architecture:
- the regions of the context tree,
- the canonical node types (and how user-assigned types fit),
- placement rules (offsets & depth),
- and the ordering contract that governs traversal and rendering.

---

## 1. Context Tree
History: the committed sequence of past episodes. Use `@t` to address time.

All PACT implementations MUST maintain a single rooted tree of context components.

### 1.1 Root
- **`^root`** — the root container (never pruned or expired).  
  All regions exist beneath it.

### 1.2 Regions (roots)
The root has three required children (regions):

- **`^sys` (alias) — System container; equals `depth(-1)`**  
  Persistent system-level instructions/background.  
  Nodes here commonly have `ttl=None`.

- **`^seq` (alias) — Conversation Sequence (structural)**  
  Structural region containing historical segments. History selection uses `@t…`; there is no mapping of history to depth.

- **`^ah` (alias) — Active Head; equals `depth(0)`**  
  Structural anchor for the working set in the current cycle.

#### Mutability Note (Normative)
- Region anchors (`^sys`, `^seq`, `^ah`) are structural only and DO NOT imply immutability. Immutability applies only to the sealed context (snapshots addressed by `@t-1`, `@t-2`, …).

### 1.3 Canonical Primitives (minimal set)
PACT components are **spatial** within a selected time. Implementations MUST support these primitives:

- **Identity** — opaque stable `id` per node (see §5 and 02 – Invariants §3)
- **Payload** — the content-bearing body (primarily in `.block` under `.cont[offset=0]`)
- **Depth** — structural region/path indicator (never temporal)
- **Offset** — relative placement within a turn: `<0` pre, `0` core, `>0` post
- **Cadence** — positive integer materialization frequency per episode (renamed from “cycle” field for refresh intervals)
- **TTL** — time-to-live in episodes (commit boundaries)

Defaults (unless explicitly set on creation):
- `ttl = null` (no TTL-based expiry)
- `cadence = 1` (fires every episode)

Notes:
- Depth/Offset govern spatial placement; Cadence/TTL govern lifecycle.
- “Episode” refers to the commit interval defined in 03 – Lifecycle; TTL decrements at commit.

### 1.4 Keys
Keys identify logical families of instances:

- **`id`** — system‑unique identifier (UUID/hash); immutable; used for provenance/audit. See 02 – Invariants §3.1.
- **`key`** — developer‑assigned family handle. A `key` MAY have multiple live instances within a snapshot due to temporal overlap (cadence/TTL). `key` stability spans episodes; instances come and go.
- **`type`**, **`tag`** — classification and labels; unchanged. Engines MUST NOT inject auto‑tags.

Notes:
- Uniqueness/invariance applies to `id`.
- Determinism for selecting among instances sharing a `key` is provided by ordering (see 02 – Invariants §3.6 and §4).

---

## 2. Node Types

PACT defines **three canonical node types** for interoperability and selectors:

- **`seg` (Segment)**  
  A structural segment under `^seq`.  
  Anchors exactly one `cont` (core) and may contain pre/post blocks.

- **`cont` (Container)**  
  The **core container** within a segment/turn. Canonical placement is defined in [02 – Invariants §4.2](02-invariants.md).  
  Holds the central message content (e.g., user request or provider reply).

- **`block` (Block)**  
  Leaf/content node: text, tool call, tool result, media, document, etc.  
  Provider‑agnostic. Attributes commonly used for interoperability include `kind`, `key`, and `tag`.

### 2.1 User‑Assigned Types (allowed and encouraged)
Implementations and developers MAY define additional **user‑assigned node types** to express domain semantics.

- **Namespace required** to avoid collisions, e.g.:
  - `block:summary`, `block:sql_query`, `block:image_caption`, `custom:note`
- **Selectors**:
  - Broad match via `.block`
  - Specific match via `[nodeType="block"][kind="summary"]` (or any user-defined value)
- **Requirement**: The canonical mapping MUST remain available so cross‑implementation queries like `.block` still work.

> PACT v0.1 has **no overlay primitive**. “Overlay‑like” annotations (summaries, corrections, redactions) are represented as `block` nodes (e.g., `nodeType="block", kind="summary"`) typically placed in **post‑context**.

---

## 3. Placement within Components
### 3.1 Canonical Offsets (at every container boundary)
Offsets apply at every container boundary (segments, containers, groups):

- `offset < 0` → pre‑context
- `offset = 0` → core container (exactly one at `offset=0`; multiple cores are invalid)
- `offset > 0` → post‑context

### 3.2 Global Depth Axis (Structural)
Depth is a single, signed axis across regions and is purely structural:

- `d = -1` → System space (`^sys`)
- `d = 0`  → Active Head (`^ah`)
- `d ≥ 1`  → Structural positions under `^seq`

Depth and any derivatives, aliases, or slices are purely structural. They MUST NEVER encode time.

### 3.3 Nested Placement Invariance
Placement and ordering rules apply recursively in every container:

- Offsets: `offset < 0` (pre) → `offset = 0` (core) → `offset > 0` (post)
- Ordering within siblings: `offset` ASC → `created_at_ns` ASC → `creation_index` ASC → `id` ASC

This nested invariance ensures a uniform ternary structure across the tree.

### 3.4 Region Aliases (Normative)
The following aliases are structural sugar over depth (also see 04 – Selectors):

- `^sys` ≡ `depth(-1)`
- `^ah`  ≡ `depth(0)`
- `^seq` — structural region for historical segments (no history↔depth mapping)

### 3.5 Structural Equivalence (Normative)

For any selector `S`, replacing `^ah` → `depth(0)` MUST select identical nodes. History selection uses `@t…` and is not encoded by depth.

### 3.6 Permissions Note

Declaring `^sys ≡ depth(-1)` does not grant user access; authorization is orthogonal. A selector may be valid yet unauthorized.

### 3.5 System Region — Canonical Layout

```
^sys
├─ prelude…   (offset=-2,-1)
├─ SYS_CORE   (offset=0)    # pinned, high TTL
└─ annexes…   (offset=+1,+2)
```

### 3.6 TTL and Cadence (Summary)
- TTL is measured per commit (episode). Default is ∞ unless explicitly set.
- Cadence controls re‑materialization per episode.
- At commit, depth increments, TTL decrements, and expired nodes drop.

### 3.7 Formal Statement
Placement invariants apply globally (depth axis) and locally (offset axis). Context is a recursive ternary structure where every container applies the same rules. Determinism is enforced by ordering, and selectors MUST resolve consistently across depth and offset.

---

## 4. Ordering (canonical, total)

Canonical sibling ordering is defined normatively in [02 – Invariants §4.3](02-invariants.md). This chapter defers to that section.

---

## 5. Lifecycle Metadata (required fields)

Every node MUST carry at least:

- `id` — opaque stable identifier (hash or UUID, impl‑defined)
- `nodeType` — canonical or user‑assigned type (e.g., `block`, `block:summary`)
- `offset` — integer (relative placement)
- `ttl` — `None | 0 | N` (cycles)
- `priority` — integer (for pruning order)
- `cycle` — introducing commit’s cycle (for this node version)
- `created_at_ns` — monotonic nanosecond timestamp
- `created_at_iso` — ISO8601 timestamp (human‑readable)
- `creation_index` — monotonic per‑cycle tie‑breaker (0,1,2,...)

---

## 6. Structural Patterns

### 6.1 Segment pattern


seg (Segment)
├─ block (offset <0) ← pre-context
├─ cont (offset =0) ← core container
│ └─ block ← core content
└─ block (offset >0) ← post-context

### 6.2 Root layout
^root
├─ ^sys (System Header)
├─ ^seq (Historical segments — structural region)
└─ ^ah (Active head: current unsealed segment)
---

## 7. Commit Model

- At each cycle’s end, the context is sealed into a snapshot (sealed context).  
- `@t0` is the active (unsealed) context; sealed snapshots are addressed by `@t-1`, `@t-2`, …  
- Fresh-@t0 equivalence: If no writes occur after the latest commit, `@t0 ≡ @t-1`. Any write breaks the equivalence; the next commit materializes a new sealed snapshot at `@t-1`.  
- Rendering has no side effects: identical snapshots → identical provider‑thread bytes.

---

- [02 – Invariants](02-invariants.md): placement, ordering, lifecycle, pruning, commit rules.