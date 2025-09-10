[← 02-invariants](02-invariants.md) | [↑ Spec Index](../README.md) | [→ 04-selectors](04-selectors.md)

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

---

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

---

## 6.1 Summarization/Rebuild (Non‑Normative)

Implementations MAY rebuild the current turn each cycle by adding new summarization/redaction/correction nodes and retiring heavier originals via TTL/budget. This enables different runtime strategies without changing sealed records or the PACT rendering rules.

---

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

→ At commit, cb expires.
→ mc is now empty; if marked removable, mc is removed too.
→ mt remains, with empty mc.

### 8.3 Sealing
^ah
├─ mc (offset=0)
│ └─ cb (role="user")
└─ cb (offset=+1, role="tool")

→ At commit, ^ah is sealed into ^seq as new mt.
→ mc core is immutable from this point onward.
→ If any removable containers become empty due to TTL expiry, they are removed in this commit (see 02 – Invariants §5.3.1).
---

[← 02-invariants](02-invariants.md) | [↑ Spec Index](../README.md) | [→ 04-selectors](04-selectors.md)
