# PACT Diagrams

This document collects visual representations of PACT concepts.  
Each diagram is referenced by the relevant spec chapter.

---

## 1. Context Tree Layout
```mermaid
graph TD
  root(^root)
  sys(^sys)
  seq(^seq)
  ah(^ah)

  root --> sys
  root --> seq
  root --> ah

```
---

## 2. Message Turn Pattern

```mermaid
graph TD
  seg(seg: MessageTurn)
  pre(block: pre-context)
  cont(cont: core container)
  post(block: post-context)

  seg --> pre
  seg --> cont
  seg --> post
  cont --> core(block: core content)
```
---
## 3. Lifecycle Flow
```mermaid
sequenceDiagram
  participant Cycle
  participant TTL
  participant Prune
  participant Seal
  participant Snapshot

  Cycle->>TTL: expire nodes
  TTL->>Prune: enforce budgets
  Prune->>Seal: move ^ah → ^seq
  Seal->>Snapshot: commit stable tree

```
---

## 4. Selector Traversal
```mermaid
graph LR
  selector["ctx.select(@t0 ( .seg .block ))"]
  snapshot["@t0 snapshot"]
  nodes{{Matched nodes}}

  selector --> snapshot --> nodes

```
---

### How to reference in spec
Instead of embedding Mermaid in every spec file, you can link to the relevant section of `docs/diagrams.md`. For example:  

```markdown
See [Diagrams: Message Turn Pattern](../docs/diagrams.md#2-message-turn-pattern).
```
