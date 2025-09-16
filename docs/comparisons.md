# Comparisons with Related Systems

PACT can be contrasted with other approaches to LLM memory and context engineering.  
This section summarizes differences between PACT and notable prior systems.

---

## MemGPT
- **Approach**: Summarization and compaction strategies to fit within token limits.  
- **Strengths**: Efficient at reducing context size, widely cited.  
- **Limitations**: Heuristic, lossy, non-deterministic ordering.  

**PACT**:  
- Defines explicit structure (`seg`, `cont`, `block`) with placement invariants.
- Deterministic lifecycle (TTL, pruning, sealing).  
- Provides selectors and diffs for reproducibility.  

---

## Letta / MemTree
- **Approach**: Hierarchical or clustered memory structures, often semantic-driven.  
- **Strengths**: Better organization than flat logs, useful for retrieval.  
- **Limitations**: No universal placement rules, pruning and ordering are heuristic.  

**PACT**:  
- Canonical placement: offset, depth, canonical sibling order.  
- Deterministic pruning: TTL → priority → age → id.  
- Byte-stable snapshots across runs.  

---

## Sculptor (2025)
- **Approach**: Structured memory to empower LLMs in tool use.  
- **Strengths**: Structured, integrates with tool-driven workflows.  
- **Limitations**: Agent-centric, model-led control of context, less universal substrate.  

**PACT**:  
- Defines a substrate independent of model strategies.  
- Provider-agnostic rendering rules.  
- Selectors enable external auditing and reproducible debugging.  

---

## Summary
- **MemGPT** → Content-first (summarization).  
- **Letta/MemTree** → Clustering-first (semantic grouping).  
- **Sculptor** → Agent-first (tool-driven structuring).  
- **PACT** → Structure-first (substrate, invariants, selectors, diffs).
