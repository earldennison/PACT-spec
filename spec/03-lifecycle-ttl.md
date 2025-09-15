[← 02-invariants](02-invariants.md) | [↑ Spec Index](../README.md) | [→ 04-selectors](04-selectors.md)

# 03 – Lifecycle and TTL

> **Terminology Clarifier — `^ah` vs `at`**  
>
> | Term | Nature | Meaning |  
> |------|--------|---------|  
> | **Active Head (`^ah`)** | Structural | The container node for the in-progress segment. On commit, ^ah becomes a seg under ^seq. Structural only; mutability is governed by the Active Turn (`at`). During `at`, nodes under depth(0) are editable subject to invariants. |  
> | **Active Turn (`at`)** | Temporal | The editable scope of the current cycle. Governs mutability across regions until sealing. |
>
> **Terminology Lint (non-normative)**  
> When writing spec text, refer to **`^ah`** for structure and **`at`** for editability.  
> Phrases like “`^ah` is editable/mutable” SHOULD be avoided; prefer  
> “during the Active Turn (`at`), content is editable until sealing.”

> **Terminology Clarifier — Cadence vs Cycle**  
> - **Cadence**: Node‑level materialization frequency (per episode); renamed from prior drafts that used “cycle” for this concept.  
> - **Cycle/Episode**: The commit iteration index for the whole system. Header `cycle` denotes the introducing commit’s index and is distinct from `cadence`.

## 1. Purpose
This section defines the **temporal semantics** of PACT:
- how cycles are defined,
- how Time-To-Live (TTL) works,
- when expiry occurs,
- how cascading cleanup is applied,
- and how lifecycle integrates with sealing and snapshots.

These rules ensure context evolves deterministically over time.

---

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

---

## 3. TTL (Time To Live) and Cadence
### 3.1 TTL Semantics
Each node has a `ttl` field interpreted in episodes (commit boundaries):
- `null` → persistent (no TTL-based expiry)
- `0` → expires at the next commit boundary
- `N > 0` → persists for N additional episodes after creation
- `N < 0` → INVALID at evaluation time; MUST be dropped

### 3.2 Evaluation
- TTL is decremented per episode at **commit time**. TTL counts episodes after creation and is independent of depth.
- Nodes with `ttl = 0` expire at the next commit boundary and do not appear in the new snapshot.

### 3.3 Decrement vs Check
Implementations MAY store TTL as “remaining episodes” (decrement each commit) or as “expiry episode” (check current episode against expiry).  
Both are valid if semantics are preserved.

### 3.4 Cadence (renamed from “cycle” frequency)
`cadence ∈ ℕ⁺` (positive integers). The cadence controls materialization frequency of instances for a logical `key`:
- `cadence = 1` → materialize every episode
- `cadence > 1` → materialize every Nth episode
- `cadence = 0` → INVALID (reject configuration)

### 3.5 Materialization Rule
When a component’s `cadence` “fires” in an episode, the implementation MUST create a new instance for its logical `key`. Prior live instances remain until TTL expiry and MUST satisfy the Overlap Demotion Invariant (ODI) (02 – Invariants §3.6): all prior instances are placed at strictly deeper depth than the newest (`depth(prev) > depth(new)`). ODI MUST be enforced at materialization time; engines MUST auto‑demote prior instances to satisfy ODI or reject the operation (`OverlapWouldViolateODI`).

Non‑normative guidance: `ttl = 1` indicates high‑salience ephemeral content (created this episode and not retained beyond sealing).

### 3.6 Universal Depth (UD)
Depth is a single axis across regions and is purely structural:

- `d = 0` → Active Turn (`^ah`), writable during the current episode
- `d ≥ 1` → History (`^seq`), newest = `1`
- `^sys` is a structural anchor outside the depth sequence

Commit transition `t → t+1` updates depths deterministically:
- if `d ≥ 1`: `depth := depth + 1`
- if `d = 0`: becomes `d = 1` (newly sealed)
- if `d = -1`: unchanged

TTL interaction under UD:
- TTL decrements only for `d ≥ 1`.
- `ttl = 0` at `d = 0` drops the node pre‑commit.

Canonical render order (equivalent to ^sys → ^seq → ^ah):
- `depth < 0` ascending (… −2, −1) → `depth > 0` descending (… 3, 2, 1) → `depth = 0` last.

### 3.7 Lifecycle Fields Only
No special tagging fields are introduced for lifecycle. Lifecycle is expressed solely via `cadence`, `ttl`, and depth placement. Engines MUST NOT auto‑tag components.

---

## 4. Expiry and Removal
### 4.1 Expiry Timing
Nodes with `ttl=0` MUST expire immediately at the next commit.  
Nodes with `ttl=N` MUST expire exactly after N cycles.

### 4.2 Cascade Rules
If expiry empties a container that is marked removable, that container MUST also be removed in the same commit.  
Root (`^root`) and region containers (`^sys`, `^seq`, `^ah`) MUST NOT be removed.

### 4.3 Reference Safety
Nodes with live references MUST NOT be removed (expired or pruned). If liveness cannot be determined, implementations MUST preserve such nodes conservatively and MAY defer expiry across commits to satisfy this requirement.

---

## 5. Sealing & Snapshots
### 5.1 Active Head
There MUST be at most one `^ah` node per snapshot (structural anchor). `^ah` denotes **structure** only; it does not define what is editable.

### 5.2 Sealing & Snapshots
At commit, the context is sealed into a snapshot (sealed context). `@t0` is the active (unsealed) working set; sealed snapshots are addressed by `@t-1`, `@t-2`, … Sealing affects mutability; `^ah` is a structural pointer, while mutability is temporal and applies to the Active Turn (`at`).

### 5.3 Effective History
After sealing, implementations MAY add new nodes (e.g., summaries, redactions, corrections) anywhere permitted by placement rules. Implementations MUST NOT mutate sealed context; changes MUST be expressed by adding new referencing nodes. PACT does not define any semantic override among such nodes. Rendering remains purely structural and deterministic: nodes are serialized in canonical order; any higher-level interpretation (e.g., "use a summary instead of originals") is an implementation choice outside this specification. Conformance is judged solely on structural serialization; semantic use of additions (summaries, redactions, corrections) is out of scope.

Implementations MUST NOT advertise semantic precedence of summaries/corrections as part of PACT conformance.

### 5.4 Determinism
Given the same prior snapshots and the same set of non‑expired additions, the effective history serializes to the same provider‑input bytes.

### 5.5 Edit Scope
**Active Turn (`at`)** — the *temporal edit scope* of the current cycle.

- Begins at cycle start and ends at commit.  
- All nodes created during the cycle (in `^ah` or other regions) are editable until sealing.  
- After sealing, further modifications MUST be expressed as new nodes, never by mutating sealed bytes.

- Within the Active Turn (at), implementations MAY add, remove, reorder, or replace any nodes (including `mc[offset=0]` contents) across regions prior to commit, provided all structural invariants hold. PACT does not constrain developer workflows inside the active cycle; it only constrains the structure and the snapshot emitted at commit.

This clarifies that editability is governed by `at`, while `^ah` is merely the structural container
of the turn that will be sealed.

 

## 5.6 Active Turn & Commit

### Active Turn (at) Mutability
- Within the Active Turn (`depth(0)`), implementations MAY add, remove, reorder, re-parent, or replace any nodes created in the current cycle (including `mc[offset=0]`) provided all structural invariants hold. PACT does not constrain developer workflows inside the active cycle; it constrains structure and the snapshot emitted at commit.
- Writes to `^sys` (`depth < 0`) are authorization/policy-gated.
 - During `at`, all nodes at `depth ≤ 0` (including `^ah` and `^sys`) are writable subject only to structural invariants and authorization.

### Commit (Sealing)
- No sealing occurs during the Active Turn. Sealing occurs only at the commit boundary.
- On commit:
  1) `depth(0)` materializes as a new `.seg:depth(1)` in `^seq` with `cont[offset=0]` bytes fixed (immutable).
  2) All existing `.seg:depth(k)` shift to `.seg:depth(k+1)`.

### Immutability
- After commit, a `seg`’s `cont[offset=0]` bytes MUST NOT be mutated. Later changes are expressed via new nodes (e.g., overlays, redactions, summaries), not by in-place mutation of sealed bytes.

## 6. Attempts & Controls
### 6.1 Stop/Edit/Re‑Run
Within a cycle, provider calls produce attempts recorded in telemetry with statuses: running, completed, canceled, error. Attempts are not part of history. If a user stops an attempt, content within the Active Turn remains editable; the system may modify or replace content and re‑run the provider. Only upon commit is the turn sealed into ^seq and a snapshot emitted. Earlier attempts remain visible to operators for observability and audit in the control plane, but do not affect history or snapshots.

### 6.2 Attempt Record Schema
Attempt records MUST include: `attempt_id`, `cycle_id`, `status`, `started_at`, `ended_at`, `provider`, `model`, `params(hash)`, `token_in`, `token_out`, `error(optional)`.

---

## 7. Order of Lifecycle Operations
At commit, lifecycle MUST be applied in this sequence:

1. **TTL Expiry**  
   Remove all nodes whose TTL has expired. Apply cascading cleanup.
2. **Pruning/Compaction (if any)**  
   Apply pruning or compaction policies if configured (see [02 – Invariants](02-invariants.md), §7).  
3. **Sealing**  
   Seal the Active Head into `^seq` as a new `seg`.  
   Create a fresh `^ah` for the next cycle.
4. **Snapshot Commit**  
   Produce a stable snapshot (`@t0`).  

This ordering ensures reproducibility across implementations.

> **Commit Sequence (Normative)** — At each cycle's commit: 1. TTL expiry and cascading cleanup; 2. Pruning/compaction (if implemented) in canonical order; 3. Seal `^ah` into a new `mt` under `^seq`; 4. Produce snapshot `@t0`. (Conformance requires identical results for identical inputs.)

---

## 6.1 Summarization/Rebuild (Non‑Normative)

Implementations MAY rebuild the current turn each cycle by adding new summarization/redaction/correction nodes and retiring heavier originals via TTL/budget. This enables different runtime strategies without changing sealed records or the PACT rendering rules.

---

## 8. Examples

### 8.1 TTL Expiry

Cycle 10:
├─ block (ttl=0) ← expires at commit
├─ block (ttl=2) ← survives until cycle 12
└─ block (ttl=None) ← persistent



### 8.2 Cascade Cleanup
seg
└─ cont
└─ block (ttl=0)

→ At commit, block expires.
→ cont is now empty; if marked removable, cont is removed too.
→ seg remains, with empty cont.

### 8.3 Sealing
^ah
├─ cont (offset=0)
│ └─ block (role="user")
└─ block (offset=+1, role="tool")

→ At commit, ^ah is sealed into ^seq as new seg.
→ cont core is immutable from this point onward.
→ If any removable containers become empty due to TTL expiry, they are removed in this commit (see 02 – Invariants §5.3.1).
---

### 8.4 Ephemeral every episode (`cadence=1, ttl=1`) with ODI

Key `K` emits a fresh instance each episode and expires immediately after sealing:

Cycle t:
^ah  └─ block[key="K", ttl=1]

Commit t→t+1:
- Newest becomes `depth=1`.
- Prior instances (if any) shift deeper (`1→2→…`) and satisfy ODI (`depth(prev) > depth(new)`).
- `ttl` on instances decrements per episode; those that reach `0` expire and are removed.

### 8.5 Periodic refresher (`cadence=3, ttl=∞`)

Key `K` materializes at t0, t3, t6, … and never expires via TTL:

- On a firing episode, create a new instance at `depth=0` that seals to `depth=1` at commit.
- Prior copies remain and drift deeper (`2,3,…`).
- Selecting the newest instance: `@t0 ( d1, *:block[key='K'] )`.
## 9. Edge Cases & Policy

- System mutability: Writes to `depth < 0`