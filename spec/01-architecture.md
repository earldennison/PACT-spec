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

All PACT implementations MUST maintain a single rooted tree of context components.

### 1.1 Root
- **`^root`** — the root container (never pruned or expired).  
  All regions exist beneath it.

### 1.2 Regions (roots)
The root has three required children (regions):

- **`^sys` (alias) — System container; equals `depth(-1)`**  
  Persistent system-level instructions/background.  
  Nodes here commonly have `ttl=None`.

- **`^seq` (alias) — Conversation Sequence; equals `{ depth(n) | n ≥ 1 }`**  
  Sequence of historical turns (mt) produced at commit.  
  Turns are appended in order; once in ^seq, each turn’s mc[offset=0] bytes are immutable.  
  Address via aliases or canonical depth: `.mt:depth(1)` is newest, `.mt:depth(2)` is the one before, etc.

- **`^ah` (alias) — Active Turn; equals `depth(0)`**  
  Holds the current in-progress turn that will be sealed into `^seq` at commit.  
  ⚠️ **Note:** Mutability is **not** a property of `^ah` itself. Editability during the cycle is governed
  by the **Active Turn (`at`)** (see 03 – Lifecycle §5.5). Only at commit is `^ah` sealed into `^seq` as a new `mt`.

#### Mutability Note (Normative)
- Region roles (`^sys`, `^seq`, `^ah`) DO NOT imply immutability in the active cycle. Only sealed cores (`depth > 0`, `mc[offset=0]`) and committed snapshots are immutable by this spec.

### 1.3 Canonical Primitives (minimal set)
PACT components are inherently **spatiotemporal**. Implementations MUST support these primitives:

- **Identity** — opaque stable `id` per node (see §5 and 02 – Invariants §3)
- **Payload** — the content-bearing body (primarily in `.cb` under `.mc[offset=0]`)
- **Depth** — universal placement along the commit axis (see §3.2 UD)
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

- **`mt` (MessageTurn)**  
  A historical turn under `^seq`.  
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
  - Specific match via `[nodeType="cb"][kind="summary"]` (or any user-defined value)
- **Requirement**: The canonical mapping MUST remain available so cross‑implementation queries like `.cb` still work.

> PACT v0.1 has **no overlay primitive**. “Overlay‑like” annotations (summaries, corrections, redactions) are represented as `cb` nodes (e.g., `nodeType="cb", kind="summary"`) typically placed in **post‑context**.

---

## 3. Placement within Components
### 3.1 Canonical Offsets (at every container boundary)
Offsets apply at every container boundary (turns, containers, groups):

- `offset < 0` → pre‑context
- `offset = 0` → core container (exactly one at `offset=0`; multiple cores are invalid)
- `offset > 0` → post‑context

### 3.2 Global Depth Axis (UD)
Depth is a single, signed axis across all regions:

- `d = 0` → Active Turn (`^ah`), writable during the current episode
- `d ≥ 1` → Sealed history (`^seq`), newest = `1`, then `2`, ... oldest
- `d = -1` → System space (`^sys`), immutable by default
- (Reserved) `d ≤ -2` → reserved for future system strata; userland MUST NOT use without explicit extension

Commit transition `t → t+1` updates depths deterministically:
- if `d ≥ 1`: `depth := depth + 1`
- if `d = 0`: becomes `d = 1` (newly sealed)
- if `d = -1`: unchanged (system)

TTL interaction under UD:
- TTL is evaluated at commit. TTL decrements only for `d ≥ 1`.
- If `ttl = 0` at `d = 0`, the node MUST be dropped pre‑commit.

Canonical render order (equivalent to ^sys → ^seq → ^ah):
- `d = -1` first → `d ≥ 1` descending (… 3, 2, 1) → `d = 0` last.

### 3.3 Nested Placement Invariance
Placement and ordering rules apply recursively in every container:

- Offsets: `offset < 0` (pre) → `offset = 0` (core) → `offset > 0` (post)
- Ordering within siblings: `offset` ASC → `created_at_ns` ASC → `creation_index` ASC → `id` ASC

This nested invariance ensures a uniform ternary structure across the tree.

### 3.4 Region Aliases (Normative)
The following aliases are pure syntactic sugar over depth (also see 04 – Selectors):

- `^sys` ≡ `.mt:depth(-1)`
- `^ah`  ≡ `.mt:depth(0)`
- `^seq` ≡ the set `{ .mt:depth(n) | n ≥ 1 }`

### 3.5 Alias≡Depth Equivalence Law (Normative)

For any selector `S`, replacing `^ah` → `depth(0)`, `^sys` → `depth(-1)`, and any `^seq` specific reference → `.mt:depth(n)` for some `n ≥ 1` MUST select the identical node(s). Implementations MUST validate this equivalence.

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
- At commit: `depth` increments (`0→1`, `1→2`, …), TTL decrements for `depth>0`, and expired nodes drop. If `ttl=0` at `depth=0`, the node drops pre‑commit.

### 3.7 Formal Statement
Placement invariants apply globally (depth axis) and locally (offset axis). Context is a recursive ternary structure where every container applies the same rules. Determinism is enforced by ordering, and selectors MUST resolve consistently across depth and offset.

---

## 4. Ordering (canonical, total)

Canonical sibling ordering is defined normatively in [02 – Invariants §4.3](02-invariants.md). This chapter defers to that section.

---

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

---

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

---

- [02 – Invariants](02-invariants.md): placement, ordering, lifecycle, pruning, commit rules.