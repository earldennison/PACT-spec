[← 03-lifecycle-ttl](03-lifecycle-ttl.md) | [↑ Spec Index](../README.md) | [→ 05-snapshots](05-snapshots.md)

# 04 – Context Queries

## 1. Purpose
This section defines the **context query language** (`ctx.select`) for querying nodes in a PACT tree.  
Queries allow developers to traverse both **space** (tree structure) and **time** (snapshots).  

Queries MUST be deterministic and side-effect free.

---

## Short Reference (one-liner to insert near top)

“Selectors: `@time` optional → `dN` depth-scope optional → `.` type anchors / `#key` / `+tag` → combinators: `>` child, space descendant → property filters `{...}`, behavior `[...]`, controls `@...`. Coordinates `(dK,...)` are canonical addresses.”

## 2. Query Model
### 2.1 Input
- A selector string (PACT query syntax).  
- An optional snapshot reference (`@t0`, `@t-1`, `@c42`).  

### 2.2 Output
- **Non-range** (single snapshot `@tN` / `@cN` / omitted ⇒ `@t0` / wildcard `@*`):  
  `ctx.select` MUST return an **ordered list of node IDs** in canonical sibling order.
- **Range present** (`@tA..@tB` or `@tA:@tB`):  
  `ctx.select` MUST return a **RangeDiffLatestResult** (pairwise diffs, snapshots sorted DESC: newest→oldest).

**Rationale.** Ranges imply evolution over time; the natural result is “what changed” rather than a flat union of IDs. This preserves stable, pre-existing behavior for non-range calls (including `@*`).

### 2.5 Canonical Selector Shape (Normative)

Canonical order for selectors:

```
@time  (position)  { attributes }  [ behavior ]  @controls
```

- Time Prefix: Canonical form uses an explicit `@time`. Parsers MUST interpret omitted time as `@t0`. Use `@t0` (active, unsealed) or `@t-1`, `@t-2`, … (sealed contexts); ranges allowed via `@tA..@tB` or `@tA:@tB`. `@history` is shorthand for `@t0..@t-*`.
- Position uses hops inside parentheses. Single-hop shorthand MAY be allowed in implementations that previously shipped it; canonical remains parenthesized.
- Insertion sentinel is a trailing comma in position; sugars: `end`, `+N` (after child N), `-N` (before child N).

---

## 3. Syntax Elements
### 3.1 Roots
- `^sys` (alias) — System container; equals `depth(-1)`  
- `^seq` (alias) — Conversation Sequence; equals `{ depth(n) | n ≥ 1 }`  
- `^ah` (alias) — Active Turn; equals `depth(0)`  
- `^root` — Root container

### 3.2 Types
- `.seg` — Segment  
- `.cont` — Container  
- `.block` — Block  

Type Sugar (Normative):
- Input sugar: `.Base:TypeName` ≡ `[nodeType='TypeName']`
  - Examples: `.cont:Message` ⇒ `[nodeType='Message']`; `.block:DiffChunk` ⇒ `[nodeType='DiffChunk']`
- Input without sugar: `.cont` ⇒ `[nodeType='cont']`, `.block` ⇒ `[nodeType='block']`, etc.
- Output normalization MUST expand sugar to attributes and MUST NOT emit `.Base:TypeName`.

Kind is not targeted by this sugar:
- `kind` is optional metadata and MUST be addressed explicitly with attributes. This specification does not use `kind` in examples.
- `.block:summary` is invalid as a `kind` shorthand and MUST be treated as TYPE sugar; reject unless `summary` is a real node type per parser policy (see Parser Behavior).

Parser Behavior (Normative options):
- If the colon suffix token resolves to a known type in the engine’s registry, parse as `nodeType`.
- If unknown and no dynamic type system is present, either:
  - treat as a parse error; or
  - accept optimistically as `[nodeType='<token>']` per implementation policy.
- Engines MUST NOT silently reinterpret `:<token>` as `kind`.

### 3.3 Keys and IDs
- `#<key>` — shorthand for `{ key="…" }` (NEVER id)
- `{ id="…" }` — select by node ID (opaque string)

### 3.4 Predicates and Depth (Structural Only)
- `:pre` — nodes with `offset < 0`  
- `:core` — nodes with `offset = 0`  
- `:post` — nodes with `offset > 0`  
- Depth denotes region only (`^sys`, `^ah`, `^seq`). It never encodes time. Use `@t…` for temporal addressing.  
  
- `:first` — first sibling among a set  
- `:last` — last sibling among a set  
- `:nth(n)` — nth sibling (1-based index)

Offset-based predicates map to canonical offset semantics defined in [02 – Invariants §4.1–4.2](02-invariants.md).

Region Aliases (Normative, pure sugar):
- `^sys` ≡ `depth(-1)` (structural)
- `^ah`  ≡ `depth(0)`  (structural)
- `^seq` — structural region for historical segments (no temporal equivalence; use `@t…` for history)

Deprecated Depth Qualifier (input-only translation): see above.

Examples (canonical forms):
- `depth(0) > .cont[offset=0]`            # active head core container
- `@t0 .seg > .block#summary`             # last committed segment’s summary (temporal)
- `@t-1 .seg`                             # an older segment (temporal)
- `depth(-1) > .block#policy`             # system container child (if authorized)

Where an alias appears, structural depth forms MUST be accepted as identical inputs. Time is addressed solely via `@t` and is independent of depth.
- `^ah > .cont[offset=0]` ≡ `depth(0) > .cont[offset=0]`
- `^sys > *` ≡ `depth(-1) > *`

Attribute filters remain unchanged and MAY be combined with depth predicates: `[key]`, `[type]`, `[tag]`, etc. Overlap note: `[key="K"]` MAY return multiple matches; use ordering or additional filters to choose the newest instance.

### 3.5 Attributes
Selectors MUST support attributes:  
`[offset] [ttl] [cad] [priority] [cycle] [created_at_ns] [created_at_iso] [nodeType] [id] [kind]`

#### 3.5.1 Grouped Shorthand (Normative)
In addition to `[attr]` filters, implementations MUST support a grouped shorthand form using parentheses immediately after a type token. Example:
```
.block(status='important' ttl=5)
```
is semantically identical to:
```
.block[status='important'][ttl=5]
```
Attributes inside `(...)` are separated by space or comma. This is purely syntactic sugar; engines MUST normalize grouped forms to canonical bracket filters before evaluation.

Normalization Rules (Type vs Kind):
- Canonical output MUST expand `.Base:TypeName` to `[nodeType='TypeName']`.
- Canonical output MUST NOT emit `.Base:<…>` for `kind`.

Parentheses immediately following a type token are reserved for this shorthand and MUST NOT collide with predicates; predicates continue to use the `:` prefix (e.g., `:first`).

#### 3.5.2 Key Selection Semantics (Normative)
- `[key="K"]` filters by logical family handle. Multiple components MAY share the same `key` within a snapshot; results MAY include multiple instances.
- Default result ordering for same‑`key` sets MUST follow 02 – Invariants §3.6 (Determinism): primary `depth` DESC, then `created_at_ns` DESC, then `creation_index` DESC, then `id` ASC.
- Implementations MUST support stable windowing over selector results (skip/limit) to enable newest/older selection. Engines MAY expose this via API chaining (e.g., `.limit(1)`, `.skip(1).limit(1)`) or equivalent selector options.

Comparison operators: `=`, `!=`, `<`, `<=`, `>`, `>=`.
 
Implementations create attributes via node headers defined in [02 – Invariants §3] (see §3.5 “Attribute Creation and Stability”). Missing optional attributes compare as `null` per §5.5; mandatory headers MUST be populated.

#### 3.5.3 Derived Facets

Time‑derived facets apply only on snapshots (turns), not on the working state:

- `born_turn` — index of the turn where this projection was created.  
- `age` — number of turns since `born_turn` at the selected snapshot.  

Structural facets like depth are addressed by position hops (e.g., `dN`, `dA..dB`) and are independent of `@t`.

Examples:

```
@t0 (d1, *:block) [age<2]
@t0..@t-3 (d2, *:block) @mode=latest [depth=2]
```

### 3.6 Structural Hops & Optional/Fallback Operators
- Structural hops:
  - Space: descendant
  - `>`: direct child
- Optional `?` (suffix on Position): for mutations only; if the match set is empty, no-op. Reads do not use `?`.
- Fallback `??` (infix between two Position patterns): evaluate left; if empty, use right (not union). If left matches but violates required cardinality, error and DO NOT evaluate right.

### 3.7 Snapshots
Time prefixes are addressable with `@t0` (active, unsealed), `@t-1` (older, sealed), `@cN` (absolute cycle), or `@*` (all). A time prefix is required in canonical form.

Range addressing across snapshots MUST be supported with **inclusive** semantics:
- `@t-<start>..<end>` — inclusive range using double-dots  
- `@t-<start>:<end>` — inclusive range using a colon (**interchangeable** with `..`)

**Contract**
- Presence of a SnapshotRange ⇒ return **RangeDiffLatestResult**.  
- Otherwise ⇒ return **ordered list of IDs**.

Ranges, `@mode`, and time‑derived facets (see §3.5.3) are valid only on snapshots (i.e., when a `@t` is present). Validators MUST reject such facets when no `@t` is provided.

#### 3.7.1 History Sugar

`@history` is shorthand for `@t0..@t-*` (newest to oldest inclusive). Implementations MUST expand this to the equivalent temporal range.

### 3.8 Position

Hops may include:

- `dN` or `depth(N)` to navigate into `^seq` (structural depth).  
- `dA..dB` — depth range hop (inclusive), selecting nodes at any depth N where A ≤ N ≤ B. Treated as a single-hop union across those depths; `dK..dK` ≡ `dK`. Depth values are integers with a minimum of `-1` (`d-1` refers to the system anchor), otherwise `0,1,2,…`. Applies anywhere a hop is valid, including within `within(...)`. Postfix `+tag`, global `{}` and `[]`, and `@controls` apply after the union.
- `seg` to indicate a segment container explicitly.  
- `cont`, `block`, and facet filters as before.  

Examples:

```
(d0, cont[offset=0], *:block)
(d1, seg, cont[offset=0], *:block)
(d2, seg, *:block)
```

### 3.9 Insertion

Use structural cues inside the selector, not API flags:

- Trailing comma `,` = insertion point.  
- `end` = append at the end of children.  
- `+N` = insert after child N.  
- `-N` = insert before child N.  

Examples:

```
(d0, seg, cont[offset=0], )                 # insert at start
(d0, seg, cont[offset=0], end)              # insert at end
(d0, seg, cont[offset=0], +1)               # insert after child 1

### 3.10 Shorthand Semantics

- `#key` ≡ `{ key="…" }` and MUST NOT resolve to IDs.
- `+tag`:
  - At root: implies deep search: `?? { tag="…" }`. Multiple `+tag` are ANDed and SHOULD be normalized alphabetically.
  - Postfix (after a position): filter only (no deep search).
- Quoted keys are supported: `#'hero banner'`.
- Formatters SHOULD merge `{ tag=… }` with `+tag` chains and normalize.
```

### 3.7.1 Conformance for Range Results (Normative)

- When a SnapshotRange is present in a selector, the result type MUST be `RangeDiffLatestResult` by default.
- No migration or legacy result modes are defined pre‑v1.

**Examples**
```text
# Non-range (IDs)
ctx.select("@t0 ( .seg .cont > .block ) { status='ok' }")
→ ["block:a1", "block:a7", ...]

# Range (RangeDiffLatestResult)
ctx.select("@t-3..@t0 ^seq .seg .block[kind='summary']")
→ {
     "query": "@t-3..@t0 .seg .block",
     "snapshots": [ { "label":"@t0" }, { "label":"@t-1" }, { "label":"@t-2" }, { "label":"@t-3" } ],
     "diffs": [
       { "from":{"label":"@t0"}, "to":{"label":"@t-1"}, "added_ids":[], "removed_ids":["block:sum:c101"], "changed":[] },
       { "from":{"label":"@t-1"}, "to":{"label":"@t-2"}, "added_ids":["block:sum:c102"], "removed_ids":[], "changed":[] },
       { "from":{"label":"@t-2"}, "to":{"label":"@t-3"}, "added_ids":[], "removed_ids":[], "changed":[] }
     ],
     "mode": "pairwise"
   }
```

---

## 4. Grammar (EBNF)

```
selector        := temporal_opt selector_body attributes_opt behavior_opt controls_opt ;
temporal_opt    := /* WS if omitted => @t0 */ | "@t" rel | "@t" rel ".." "@t" rel | "@history" ;
rel             := "0" | ("-" uint) | ("+" uint) ;

selector_body   := coordinate
                 | coordinate_set
                 | chain_selector
                 | prop_search ;

coordinate      := "(" "d" int ("," int )* ")" ;            # (dK, o1, o2, ...)
coordinate_set  := "(" coordinate ("," coordinate)* ")" ;   # ((d1,0),(d2,0,1))

chain_selector  := scope_opt? selector_chain ;
scope_opt       := "d" int ( ".." int )? ;                   # optional depth scope (d1 or d1..d8)
selector_chain  := simple_sel ( ws combinator ws simple_sel )* ;
simple_sel      := anchor ( "." ident )* ;                   # .summary, .note, .mc, .seg, .cont, etc.
anchor          := "." ident   # type or structural (.mc, .seg, .cont, .sys, .summary, .note)
                 | "#" key_token
                 | "+" ident ;                              # root tag anchor

combinator      := ">" | WS ;                                # child or descendant

prop_search     := "{" prop ("," prop)* "}" ;                # standalone prop search

attributes_opt  := ( ws* "{" prop ( "," prop )* "}" )? ;
behavior_opt    := ( ws* "[" bprop ( "," bprop )* "]" )? ;
controls_opt    := ( ws+ control )* ;

prop            := ident cmp value | ident "=" array | ident "=" value ;
bprop           := ident bcmp bval | "ephemeral" | "sticky" ;
cmp             := "=" | "!=" | ">=" | "<=" | ">" | "<" | "~" ;
bcmp            := "=" | "!=" | ">=" | "<=" | ">" | "<" | "~" ;
value           := int | ident | quoted ;
array           := "[" value ( "," value )* "]" ;
control         := "@" "*" | "@" ident ( "=" ident )?      # allow @* and vendor @foo or @limit=50 etc.
int             := ("-"|"+")? uint ;
uint            := DIGIT { DIGIT } ;
quoted          := "'" { ANY_BUT_SINGLE_QUOTE | "\\'" } "'"
                 | '"' { ANY_BUT_DOUBLE_QUOTE | '\\"' } '"' ;
```

### 4.3 Deletions (remove these productions entirely)

* `hop`, `deep_search`, `within`, `shorthand_root`, `hop_propblock`, `postfix_tags_opt`.
* Any alternatives referencing: `"cont" "[" "offset" "=" int "]"`, `"seg" (...)`, `("*@+" | "*@-")`, or child-combinator forms.

## 5. Selector Semantics (Overview)

- **. = type anchor**: Examples: `.summary`, `.note`, `.cta`. Structural anchors `.mc`, `.seg`, `.cont`, `.sys` remain reserved.
- **combinators**: `>` = direct child; space = descendant (any depth).
- **coordinates**: `(dK, offset1, offset2, ...)` and coordinate sets `((...),(...))` remain canonical.
- **scope prefix**: `dN` or `dA..dB` allowed as shorthand for limiting selector evaluation to those depths.
- **attributes**: `{ ... }` = node property filters (works standalone as deep-search too).
- **behavior**: `[ ... ]` = behavior/lifecycle (e.g., `ttl`, `cad`, `ephemeral`, `sticky`).
- **controls**: `@...` = controls, allow vendor passthrough via `@*`.
- **time**: Default time lens is `@t0` when omitted.

---

## 6. Temporal Lens

* **Default:** If time is omitted, interpret as `@t0`.
* Keep `@tN`, `@tA..@tB`, and `@history` as-is.

---

## 6.2 Canonical Selector Tests (Normative)

#### Range (returns RangeDiffLatestResult)
- MUST return `RangeDiffLatestResult` for any selector containing a SnapshotRange.
- Snapshots MUST be sorted newest→oldest (DESC by cycle) in the `snapshots` array.
- `diffs` MUST be pairwise: (s0→s1), (s1→s2), …

#### Non-range (returns IDs)
- MUST return ordered array of node IDs for single-snapshot or `@*` queries.

Given this minimal snapshot fixture at `@t0`:

```json
{
  "spec_version": "PACT/1.0.0",
  "root": {
    "children": [
      { "id": "sys-1", "nodeType": "^sys", "children": [
        { "id": "block:sysA", "nodeType": "block", "kind": "text", "offset": 0, "content": "S" }
      ]},
      { "id": "seq-1", "nodeType": "^seq", "children": [
        { "id": "seg:1", "nodeType": "seg", "children": [
          { "id": "cont:1", "nodeType": "cont", "children": [
            { "id": "block:u1", "nodeType": "block", "kind": "text", "offset": 0, "ttl": 2, "content": "U1" }
          ]},
          { "id": "cont:2", "nodeType": "cont", "children": [
            { "id": "block:a1", "nodeType": "block", "kind": "text", "offset": 0, "ttl": 1, "content": "A1" }
          ]}
        ]},
        { "id": "seg:2", "nodeType": "seg", "children": [
          { "id": "cont:3", "nodeType": "cont", "children": [
            { "id": "block:u2", "nodeType": "block", "kind": "text", "offset": 0, "content": "U2" }
          ]}
        ]}
      ]},
      { "id": "ah-1", "nodeType": "^ah", "children": [
        { "id": "block:u3", "nodeType": "block", "kind": "text", "offset": 0, "content": "U3" }
      ]}
    ]
  }
}
```

Implementations MUST produce exactly these results (IDs and order):

1. `@t0 ^sys .block` → `["block:sysA"]`
2. `@t0 .seg` → `["seg:1"]`
3. `@t0 .seg` and `@t-1 .seg` (evaluated separately) → `["seg:1"]`, `["seg:2"]`
4. `@t0 .seg .cont > .block` → `["cont:1","cont:2"]`
5. `@t0 #block:u2` → `["block:u2"]`

## 6.3 Additional Range Fixture (Normative)

To validate range depth selectors, the following minimal snapshot fixture at `@t0` MUST produce the listed results:

```json
{
  "spec_version": "PACT/1.0.0",
  "root": {
    "children": [
      { "id": "seq-2", "nodeType": "^seq", "children": [
        { "id": "seg:1", "nodeType": "seg", "children": [ { "id": "cont:1", "nodeType": "cont", "children": [ { "id": "block:u1", "nodeType": "block", "kind": "text", "offset": 0, "content": "U1" } ] } },
        { "id": "seg:2", "nodeType": "seg", "children": [ { "id": "cont:2", "nodeType": "cont", "children": [ { "id": "block:u2", "nodeType": "block", "kind": "text", "offset": 0, "content": "U2" } ] } },
        { "id": "seg:3", "nodeType": "seg", "children": [ { "id": "cont:3", "nodeType": "cont", "children": [ { "id": "block:u3", "nodeType": "block", "kind": "text", "offset": 0, "content": "U3" } ] } }
      ]}
    ]
  }
}
```

- Query (evaluate per snapshot): `@t0 .seg .block`, `@t-1 .seg .block`, `@t-2 .seg .block`
  Expect: per-snapshot IDs in canonical order; cross-snapshot recency is expressed by the `@t…` prefix.


## 8. Canonical Coordinates

* Define coordinate precisely: `(dK, offset₁, offset₂, …)` with `d0` = active head, `d1` = latest sealed, `d-1` = system.
* Offsets are integers; `0` is the core child within a turn (`mc`).
* Coordinates are unique within a snapshot; across episodes, depth indices re-derive.

## 9. Examples (replace with minimal, compliant set)

Examples use the chained selector syntax (dot-chains and `>` for child-of). Coordinates and property-search forms are also shown.

- `@t0 (d1,0,1)` — exact coordinate
- `@t0 ((d1,0), (d2,0,1))` — coordinate set
- `@t0 d1..d3 .summary` — summaries in depths 1–3
- `.mc > .container > .section > .cta { variant in ["primary","ghost"] }`
- `.summary .note { author="assistant" } @limit=10`
- `#nav .item`
- `+audit .summary [ ttl>=2 ] @limit=50`
- `(d2,0) .note { lang="en" }`
- `{ nodeType="block", type="summary", author="user" }`
- `d0 .config { key="theme" }`
- `@history d1..d5 .log.entry { severity="P1" } [ ttl>=1 ] @order=path @limit=20`

## Migration Recipe

### Regex replace (git-run friendly examples)

```text
s/\.block:([A-Za-z0-9_]+)/.\1/g   # `.block:summary` → `.summary`
s/cont\[offset=([0-9-]+)\]/\1/g    # `cont[offset=0]` → `0`
s/c@([0-9]+)/ (\1)/g                # best-effort cleanup for c@→ coordinate form (manual verify)
s/\*\@[\+\-]/[use-offset-filter]/g # remove \*@+/- occurrences; replace with offset filters
```

### Linter rules

- Flag any `cont[...]`, `c@`, `.block:...`, `*@+/-` uses as errors.
- Auto-fix `.block:TYPE` → `.TYPE`.
- Auto-fix `cont[offset=N]` → `N` inside coordinates/paths.

Add note: run migration branch + tests; update examples and unit tests.

## 10. Conformance & Validation

- Parentheses parse only to coordinate or coordinate set (unless explicitly using `chain_selector`).
- `.TYPE` anchors must map to nodes whose `type==TYPE`.
- If `ttl`/`cad` are found in `{...}`, validator MUST recommend moving them to `[...]`.
- Any removed legacy tokens cause a validation error.
- Validation: any occurrence of the token "CSS" or explicit DOM/CSS comparisons in headings/subheadings should be flagged and removed; authors must use neutral phrasing such as "chained selector syntax" or "selector chains".

[← 03-lifecycle-ttl](03-lifecycle-ttl.md) | [↑ Spec Index](../README.md) | [→ 05-snapshots](05-snapshots.md)

