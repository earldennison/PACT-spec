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

- **`^sys` — System Header**  
  Persistent system-level instructions/background.  
  Nodes here commonly have `ttl=None`.

- **`^seq` — Conversation Sequence**  
  Sequence of **sealed turns** (`mt`).  
  Turns are appended in order; once sealed, their **core** is immutable.

- **`^ah` — Active Head**  
  The current, in‑progress turn (depth 0).  
  Mutable until sealed into `^seq`. At most one active head exists per snapshot.

---

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

---

## 3. Placement
### 3.1 Relative Offsets
Canonical placement (offset semantics and `mc@0`) is defined normatively in [02 – Invariants §4.1–4.2](02-invariants.md). This chapter is non‑normative for placement.

### 3.2 Depth (sealed sequence addressing)
Depth addresses sealed turns in `^seq` from newest to oldest:

- `depth = 1` → most recent sealed turn  
- `depth = 2` → the turn before it, etc.  
- `^ah` (active head) is not depth‑addressable; it is addressed by the root alias.

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
└─ ^ah (Active head: current, mutable turn)
---

## 7. Commit Model

- At each cycle’s end, the Active Head (`^ah`) is **sealed** into a new `mt` under `^seq`.  
- A fresh Active Head is created for the next cycle.  
- A **snapshot** is produced at commit; rendering has no side effects: same snapshot → identical provider‑thread bytes.

---

## Next
- [02 – Invariants](02-invariants.md): placement, ordering, lifecycle, pruning, commit rules.