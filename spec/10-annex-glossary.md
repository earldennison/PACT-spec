[← 09-annex-conformance](09-annex-conformance.md) | [↑ Spec Index](../README.md)

# 11 – Annex: Glossary (Non‑Normative)

Concise definitions of key PACT terms, with section references.

- Cycle: One rebuild+commit iteration. Exactly one snapshot per cycle. See 03 – Lifecycle §2.
- Snapshot: The frozen picture of the entire tree taken at commit. Snapshots are immutable and one‑to‑one with turns (every turn is a snapshot). A snapshot fixes the state of all segments in `^seq`, the system container, and the active head at commit time. See 05 – Snapshots §2.1.
- Sealing (operation): The commit-time action that moves the in-progress turn from ^ah into ^seq and fixes the bytes of its mc[offset=0] core. See 02 – Invariants §6.2, §6.4.
**Active Head (`^ah`)** — Working state container at `d0` (editable). At commit, `^ah` becomes a new segment appended to `^seq` in the new turn.
**Active Turn (`at`)** — Temporal scope of editability within the current cycle.
Includes the `^ah` and any other nodes created during this cycle. Ends when commit seals the turn.
- Turn: A committed snapshot of the entire context tree. Each commit produces exactly one new turn. Immutable. Contains `^sys`, `^seq` (segments), and `^ah` as it was at commit.
- Segment (`seg`): Structural container under `^seq` holding exactly one `cont` core (`offset=0`) plus optional pre-context (`block` with `offset<0`) and post-context (`block` with `offset>0`). At commit, `d0` becomes a new segment appended to `^seq` in the new turn. Segments are structural only; they carry no temporal meaning.
- Active Head (^ah): The working state at depth(0). Fully mutable subject to structural invariants until commit; becomes a segment at commit.
- Region Aliases: ^ah (alias) ≡ depth(0), ^seq (alias) ≡ { .seg:depth(n) | n≥1 }, ^sys (alias) ≡ depth(-1).
- Terminology: Do not use “sealed conversational turn.” Use “turn” only for the full snapshot. Use “segment” for the structural container under `^seq`. Use “sealing” only to refer to the commit-time operation.
- Core (`cont`): Container at `offset=0` inside a segment/turn; the main message body. Exactly one per segment. See 02 – Invariants §4.2.
- Block (`block`): Leaf/content node (text/call/result/media, etc.) with optional `role` and `kind`. See 01 – Architecture §2.
- Combinator: Selector relationship operators. Descendant (`A B`) matches any descendant; child (`A > B`) matches direct children only. See 04 – Selectors §3.3, §4.
- Effective History: Rendered history computed from historical turns plus any additions
  (e.g., summaries, redactions, corrections) created after sealing. **PACT does not
  define any semantic precedence or override among such additions**; rendering is
  purely structural and deterministic. Higher-level interpretation (e.g., “use a
  summary instead of originals”) is an implementation decision outside PACT
  conformance. See 03 – Lifecycle §5.3.lso
- Depth: Structural index inside `^seq` within a turn: `d0` = active head, `d1,d2,…` = segments, `d-1` = system. Structural only; not a synonym for turn. See 01 – Architecture §3.2.
- Offset: Position of children inside a segment: `<0` pre (`block`), `=0` core (`cont`), `>0` post (`block`). See 02 – Invariants §4.1.
- Regions: Root children `^sys`, `^seq`, `^ah` under `^root`. See 02 – Invariants §2.
- Commit Sequence: TTL → Pruning/Compaction (if implemented) → Seal → Snapshot. Canonical rule in 03 – Lifecycle §7.
- Removable container: A container explicitly marked `removable=true` at creation or by deterministic policy; when all children expire or are pruned, it MUST be removed in the same commit. Root (`^root`) and region containers (`^sys`, `^seq`, `^ah`) MUST NOT be removable. See 02 – Invariants §5.3.1.
- Reference safety: Nodes referenced by live pointers MUST NOT be expired or pruned. When reference liveness is unknown, preserve conservatively. See 03 – Lifecycle §4.3 and 02 – Invariants §5.4.
- Canonical pruning order: If pruning is implemented, nodes MUST be pruned by `priority`, then age (`created_at_ns` ascending), then `id` (lexicographic). See 02 – Invariants §7.1.
- Canonical Order: Sibling ordering is deterministic: `offset` asc → `created_at_ns` asc → `creation_index` asc → `id` lexicographic. See 02 – Invariants §4.3.

- Key (family handle): Developer‑assigned label used to group related instances. Multiple live instances with the same `key` MAY coexist in a snapshot due to cadence/TTL overlap. Deterministic ordering among same‑`key` sets is defined in 02 – Invariants §3.6.
- Instance: A concrete component with its own `id` and optional `key`. Lifecycle (materialization and expiry) is controlled by cadence and TTL.
- Overlap Demotion Invariant (ODI): If a new instance for a `key` is materialized while prior instances are alive, all prior instances MUST be placed at strictly deeper depth than the newest (`depth(prev) > depth(new)`) under Universal Depth. Engines MUST auto‑demote or reject placements that would violate ODI. See 02 – Invariants §3.6 and 03 – Lifecycle §3.5.

- Golden Tests: Normative selector fixture results (04 – Selectors §6.2–§7.1). Implementations MUST produce exactly the listed IDs and order for each query; used for conformance.

- Attempt: Provider call execution during the Active Turn with telemetry: `attempt_id`, `cycle_id`, `status`, `started_at`, `ended_at`, `provider`, `model`, `params(hash)`, `token_in`, `token_out`, `error(optional)`. See 03 – Lifecycle §6.
- RangeDiffLatestResult: The result shape returned when a SnapshotRange is present in a selector. Contains the original `query`, resolved `snapshots` (DESC by cycle), pairwise `diffs` blocks with `added`/`removed`/`changed`, and metadata like `mode` and optional `limits`. See 04 – Selectors §3.7.

[← 09-annex-conformance](09-annex-conformance.md) | [↑ Spec Index](../README.md)


