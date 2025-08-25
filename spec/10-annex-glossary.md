[← 09-annex-conformance](09-annex-conformance.md) | [↑ Spec Index](../README.md)

# 11 – Annex: Glossary (Non‑Normative)

Concise definitions of key PACT terms, with section references.

- Cycle: One rebuild+commit iteration. Exactly one snapshot per cycle. See 03 – Lifecycle §2.
- Snapshot: Byte-stable tree after TTL and sealing; pruning/compaction is optional. `@t0` MUST re‑serialize to provider bytes. See 05 – Snapshots §2.1.
- Sealed (turn/core): A turn (`mt`) sealed from `^ah` into `^seq`. Its `mc@0` core is immutable. See 02 – Invariants §6.2, §6.4.
- Active Head (`^ah`): Current, mutable turn (depth 0). See 01 – Architecture §1.2.
- Turn (`mt`): Sealed conversational turn under `^seq`. Anchors exactly one `mc` (core) and may include pre/post `cb` children. See 01 – Architecture §2.
- Core (`mc`): MessageContainer at `offset=0` inside a turn; the main message body. Exactly one per `mt`. See 02 – Invariants §4.2.
- Content Block (`cb`): Leaf/content node (text/call/result/media, etc.) with optional `role` and `kind`. See 01 – Architecture §2.
- Combinator: Selector relationship operators. Descendant (`A B`) matches any descendant; child (`A > B`) matches direct children only. See 04 – Selectors §3.3, §4.
- Effective History: Rendered history computed from sealed turns plus any additional nodes added after sealing. PACT does not define any semantic override; rendering is structural. See 03 – Lifecycle §5.3.
- Depth: Addressing sealed turns in `^seq` newest→oldest. `depth=1` is most recent. See 01 – Architecture §3.2.
- Offset: Relative placement within a turn: `<0` pre, `0` core (`mc`), `>0` post. See 02 – Invariants §4.1.
- Regions: Root children `^sys`, `^seq`, `^ah` under `^root`. See 02 – Invariants §2.
- Commit Sequence: TTL → Pruning/Compaction (if implemented) → Seal → Snapshot. Canonical rule in 03 – Lifecycle §7.
- Removable container: A container explicitly marked `removable=true` at creation or by deterministic policy; when all children expire or are pruned, it MUST be removed in the same commit. Root (`^root`) and region containers (`^sys`, `^seq`, `^ah`) MUST NOT be removable. See 02 – Invariants §5.3.1.
- Reference safety: Nodes referenced by live pointers MUST NOT be expired or pruned. When reference liveness is unknown, preserve conservatively. See 03 – Lifecycle §4.3 and 02 – Invariants §5.4.
- Canonical pruning order: If pruning is implemented, nodes MUST be pruned by `priority`, then age (`created_at_ns` ascending), then `id` (lexicographic). See 02 – Invariants §7.1.
- Canonical Order: Sibling ordering is deterministic: `offset` asc → `created_at_ns` asc → `creation_index` asc → `id` lexicographic. See 02 – Invariants §4.3.

- Golden Tests: Normative selector fixture results (04 – Selectors §6.2–§7.1). Implementations MUST produce exactly the listed IDs and order for each query; used for conformance.

[← 09-annex-conformance](09-annex-conformance.md) | [↑ Spec Index](../README.md)


