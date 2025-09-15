# Glossary

**PACT (Positioned Adaptive Context Tree)**  
A deterministic substrate for LLM context: governed structure, lifecycle, and rendering invariants.

**Region**  
Top‑level partition of the context tree: **^sys** (system header), **^seq** (sealed sequence of turns), **^ah** (active head; convenience alias), **^root** (root container).

**Turn**  
A committed snapshot of the entire context tree.  
- Each commit produces exactly one new turn.  
- A turn is immutable.  
- A turn contains the system container (`^sys`), the sequence of structural segments (`^seq`), and the active head (`^ah`) as it was at commit.  
- Developers use “turn” only to mean the full snapshot, never an internal node.

**Segment (`seg`)**  
Structural container under `^seq`.  
- Each segment holds exactly one core (`cont`) plus optional pre-context (`block` with `offset < 0`) and post-context (`block` with `offset > 0`).  
- Segments are ordered strictly by depth index (`d1`, `d2`, …).  
- At commit, the active head (`d0`) becomes a new segment appended to `^seq` in the new turn.  
- Segments are structural only; they carry no temporal meaning.

**Active Head (`^ah`)**  
The working state container at `d0` (editable). At commit, `^ah` becomes a new segment appended to `^seq` in the new turn.

**Container (`cont`)**  
Core container at **`offset = 0`** within a segment/turn. Pre‑context attaches at negative offsets; post‑context at positive offsets.

**Block (`block`)**  
Leaf/content node (text, tool call, tool result, media, document, etc.). Provider‑agnostic. Common attributes include `kind`, `key`, and `tag`.

**NodeType**  
A string that classifies a node.
- **Canonical types**: `seg`, `cont`, `block` — required for interoperability.  
- **User‑assigned types**: Implementations or developers may define namespaced types (e.g., `block:summary`, `block:sql_query`, `custom:note`).  
Selectors MUST be able to target both canonical and user‑assigned types (e.g., `.block` or `[nodeType="block"][kind="summary"]`).

**Relative Offset (`offset`)**  
Position of children inside a segment: `< 0` = pre‑context `block`, `= 0` = `cont` core, `> 0` = post‑context `block`.

**Depth**  
Structural index inside `^seq` within a turn.  
- `d0` = active head (working state, editable).  
- `d1`, `d2`, … = the first, second, … segment inside `^seq`.  
- `d-1` = system container.  
Depth is structural only; it is not a synonym for turn.
Entering `seg` is implicit after a `dN` hop; write `seg{…}` only when filtering/operating on the segment node.

**Cycle**  
One rebuild/commit iteration of the tree. TTL expiry and snapshots align to cycles.

**TTL (Time To Live)**  
Cycles a node persists. `None` = persistent; `0` = expires after current cycle; `N` = lasts N cycles.

**Snapshot**  
The frozen picture of the entire tree taken at commit.  
- Snapshots are immutable.  
- Snapshots are one‑to‑one with turns (every turn is a snapshot).  
- A snapshot fixes the state of all segments in `^seq`, the system container, and the active head at commit time.

**History**  
The immutable timeline of snapshots. History is composed of successive snapshot records; snapshots themselves are immutable artifacts.

**Determinism (in PACT)**  
Guarantee that **context assembly and rendering are deterministic**.  
- Same snapshot → identical serialized context bytes to the provider.  
- Does **not** guarantee deterministic LLM outputs (which are stochastic).

**Canonical Order**  
Deterministic sibling ordering: `(offset asc, created_at_ns asc, creation_index asc, id asc)`.

**Creation Index (`creation_index`)**  
Monotonic per‑cycle tie‑breaker assigned at creation (0,1,2,...). Used after `created_at_ns` to ensure total ordering among siblings created in the same timestamp window.

**Removable Container**  
Container explicitly marked `removable=true` at creation or by deterministic config policy. If all children expire/prune, the container is removed in the same commit. Root and region containers are never removable.

**Budget / Pruning**  
Deterministic enforcement of resource limits (tokens/nodes/depth/region). Order: **expire TTL → prune by priority → age → id**.

**Transaction / Commit**  
Atomic update that produces a snapshot. Rendering has no side effects; same snapshot → identical provider‑thread bytes.

**Context Selectors (`ctx.select`)**  
CSS‑like query language for space (**regions/segments/offsets**) and time (**turns/snapshots**). Supports `^roots`, `#id`, `.type` (canonical or user‑assigned), combinators, pseudos, and attributes.

**Diff (`ctx.diff`)**  
Snapshot delta for a selector: `{added, removed, changed}` by node ID (field‑wise change report recommended). Structural changes SHOULD track re‑parenting explicitly: in the old parent `children_removed += id`, in the new parent `children_added += id`; implementers MAY also include a child‑level `parent_changed {previous_parent_id, new_parent_id}` signal.

**RangeDiffLatestResult**  
Result shape for range selects. Includes `query`, resolved `snapshots` sorted newest→oldest, pairwise `diffs` with `added`/`removed`/`changed`, and optional `mode`/`limits` metadata. See Selectors §3.7.

**Re‑parenting**  
Changing a node’s parent. Prior to commit, any node MAY be re‑parented except `mt` (message turns) and the region container `^sys`. After sealing, cores (`mc[offset=0]`) are immutable; non‑core nodes MAY still be attached/detached across turns via new nodes and lifecycle rules.

**Attempt**  
A provider call execution within the Active Turn. Tracked in telemetry with fields like `attempt_id`, `cycle_id`, `status`, timestamps, provider/model, params hash, token counters, and optional error. See Lifecycle §6.

**Provider Thread**  
Provider‑agnostic linearization (ClientRequest / ProviderResponse) derived deterministically from the PACT tree.

**Working State (WS)**  
The live, editable tree (no `@t`). WS has no time‑derived properties; edits and structural mutations are allowed until commit, subject to invariants and validation.

**Key vs ID**  
IDs are engine‑assigned and hidden by default; developers address nodes by `key` or structure.  
- `#key` ≡ `{ key='…' }` (NEVER id)  
- `{ id='…' }` selects by ID explicitly  
- `+tag` at root implies deep search (`?? { tag=… }`); postfix `+tag` filters only
