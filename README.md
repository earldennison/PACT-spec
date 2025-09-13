# PACT – Positioned Adaptive Context Tree

**PACT** is a specification for deterministic context engineering in large language model (LLM) systems.  
It defines a tree-structured substrate with placement, lifecycle, and pruning invariants, ensuring that
context assembly is reproducible, auditable, and provider-agnostic.

---

## Why PACT?

Most LLM systems treat context as an **ad-hoc log of messages**.  
This leads to:
- Non-deterministic results (different runs = different context orderings),
- Fragile memory pressure handling,
- Difficulty debugging or replaying state,
- Provider-specific quirks.

**PACT treats structure as a first-class concept**:
- Regions: `^sys` (system header), `^seq` (historical turns), `^ah` (active head),
- Canonical node types: `mt`, `mc`, `cb`,
- Deterministic ordering: offset → created_at_ns → creation_index → id,
- Lifecycle: cycles, TTL, sealing, cascading cleanup,
- Budgeting & pruning: reproducible resource enforcement,
- Snapshots & diffs: stable replay + debugging,
- Context Selectors: CSS-like traversal across space and time.

- Active Turn (`at`) vs `^ah`: `at` governs editability during a cycle; `^ah` is structural only.
- Snapshot addressing: `@t0`, `@t-1`, `@cN`; ranges (`..` or `:`) are inclusive and range selects return diffs.

---

## Spec

The normative specification lives in [`/spec`](./spec):

- [00 – Overview](./spec/00-overview.md)  
- [01 – Architecture](./spec/01-architecture.md)  
- [02 – Invariants](./spec/02-invariants.md)  
- [03 – Lifecycle and TTL](./spec/03-lifecycle-ttl.md)  
- [04 – Context Selectors](./spec/04-selectors.md)  
- [05 – Snapshots and Diffs](./spec/05-snapshots.md)  
- [06 – Debugging and Inspection](./spec/06-debugging.md)  
- [07 – Adoption and Compatibility](./spec/07-adoption.md)  
- [08 – Reference Implementations](./spec/08-reference-implementations.md)
- [09 – Annex: Spec-wide Conformance Checklist](./spec/09-annex-conformance.md)
- [10 – Annex: Glossary](./spec/10-annex-glossary.md)
 
The list above serves as the spec index. For a single-file view, see [`PACT-Complete-Specification.md`](./PACT-Complete-Specification.md).

---

## Documentation

- [Glossary](./docs/glossary.md) — central definitions.  
- [Comparisons](./docs/comparisons.md) — how PACT differs from MemGPT, Letta, Sculptor, etc.  
- [Diagrams](./docs/diagrams.md) — Mermaid diagrams illustrating the model.  
- [Spec Style Guide](./docs/spec-style.md) — format and tone rules for spec files.  
- [Selector Examples](./docs/selector-examples.md) — query patterns across space/time.  

---

## Version

Current spec version: **0.1.0**  
See [`VERSION`](./VERSION) for canonical version string.

---

## License

Licensed under the [Apache License, Version 2.0](./LICENSE).

---

## Citation

If you use this spec, please cite it via the [`CITATION.cff`](./CITATION.cff).
