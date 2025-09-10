[↑ Spec Index](../README.md) | [→ 01-architecture](01-architecture.md)

# 00 – Overview

## What is PACT?

**PACT (Positioned Adaptive Context Tree)** is a deterministic substrate for managing context in large language model (LLM) systems.  
Instead of treating context as an unstructured sequence of messages, PACT defines a **tree with invariants** for placement, lifecycle, and rendering.  

This enables developers to build agent frameworks and runtime systems that are **predictable, auditable, and reproducible** at the level of **context state**.

---

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

---

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

---

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

---

## Why it matters

PACT’s novelty is that it treats **context structure as a first-class substrate**, not just the content inside.  
This separation enables:

- **Determinism** — the same snapshot always renders to the same serialized provider-thread bytes, regardless of LLM randomness.  
- **Auditability** — easy to diff and replay snapshots, independent of model output.  
- **Composability** — content strategies (summaries, RAG, tools) can plug into a stable substrate.  
- **Interoperability** — provider-agnostic rendering ensures portability across vendors.  
- **Debuggability** — developers can ask “what changed?” at the context layer, regardless of stochastic LLM outputs.

---

## Next

- [01 – Architecture](01-architecture.md): defines the regions, node types, and container patterns.  
- [02 – Invariants](02-invariants.md): specifies placement, ordering, lifecycle, and commit rules.  

---

[↑ Spec Index](../README.md) | [→ 01-architecture](01-architecture.md)
