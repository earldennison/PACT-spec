[← 03-lifecycle-ttl](03-lifecycle-ttl.md) | [↑ Spec Index](../README.md) | [→ 05-snapshots](05-snapshots.md)

# 04 – Context Selectors

## 1. Purpose
This section defines the **context selector language** (`ctx.select`) for querying nodes in a PACT tree.  
Selectors allow developers to traverse both **space** (tree structure) and **time** (snapshots).  

Selectors MUST be deterministic and side-effect free.

---

## 2. Selector Model
### 2.1 Input
- A selector string (CSS-like syntax).  
- An optional snapshot reference (`@t0`, `@t-1`, `@c42`).  

### 2.2 Output
- **Non-range** (single snapshot `@tN` / `@cN` / omitted ⇒ `@t0` / wildcard `@*`):  
  `ctx.select` MUST return an **ordered list of node IDs** in canonical sibling order.
- **Range present** (`@tA..@tB` or `@tA:@tB`):  
  `ctx.select` MUST return a **RangeDiffLatestResult** (pairwise diffs, snapshots sorted DESC: newest→oldest).

**Rationale.** Ranges imply evolution over time; the natural result is “what changed” rather than a flat union of IDs. This preserves stable, pre-existing behavior for non-range calls (including `@*`).

---

## 3. Syntax Elements
### 3.1 Roots
- `^sys` (alias) — System container; equals `depth(-1)`  
- `^seq` (alias) — Conversation Sequence; equals `{ depth(n) | n ≥ 1 }`  
- `^ah` (alias) — Active Turn; equals `depth(0)`  
- `^root` — Root container

### 3.2 Types
- `.mt` — MessageTurn  
- `.mc` — MessageContainer  
- `.cb` — ContentBlock  
- Namespaced user-assigned types MUST also be supported. A type token like `.cb:summary` MUST match nodes whose `nodeType` equals `"cb"` and whose `kind` equals `"summary"`.
- Equivalence (Normative): `.cb:summary` and `[nodeType='cb'][kind='summary']` MUST be treated as equivalent forms and MUST return identical results.
- `kind` is not a type; query it via attributes (e.g., `.cb[kind='tool_call']`). Bare `.tool_call` is not standardized and MUST NOT be required for conformance.

### 3.3 IDs
- `#<id>` — select by node ID (opaque string)

### 3.4 Pseudo-classes and Depth Qualifier
- `:pre` — nodes with `offset < 0`  
- `:core` — nodes with `offset = 0`  
- `:post` — nodes with `offset > 0`  
- `:depth(expr)` — select turns by Universal Depth (03 – Lifecycle §3.6):  
  - Exact: `:depth(0)`, `:depth(1)`, `:depth(-1)`  
  - Lists: `:depth(1,2,5)`  
  - Inclusive ranges: `:depth(1-3)` or `:depth(1..3)`  
  - Comparisons: `:depth(<0)`, `:depth(>0)`, `:depth(>=2)`, `:depth(<=-1)`  
  - Sets: `:depth({1,2,4})`  
- `:first` — first sibling among a set  
- `:last` — last sibling among a set  
- `:nth(n)` — nth sibling (1-based index)

Offset-based pseudos map to canonical offset semantics defined in [02 – Invariants §4.1–4.2](02-invariants.md).

Region Aliases (Normative, pure sugar):
- `^sys` ≡ `depth(-1)`
- `^ah`  ≡ `depth(0)`
- `^seq` ≡ the set `{ depth(n) | n ≥ 1 }`

Depth may be expressed directly as a selector qualifier:
- `.mt:depth(n)` where `n ∈ ℤ`
- Optional extension: `.mt:depth(a..b)` with `a ≤ b`, inclusive

Examples (canonical forms):
- `depth(0) > .mc[offset=0]`            # active turn core
- `.mt:depth(1) > .cb#summary`          # last committed turn’s summary
- `.mt:depth(2)`                         # second most recent committed turn
- `depth(-1) > .cb#policy`               # system container child (if authorized)

Where an alias appears, the depth form MUST be accepted and treated as identical:
- `^ah > .mc[offset=0]` ≡ `depth(0) > .mc[offset=0]`
- `^seq .mt:depth(1)` ≡ `.mt:depth(1)`
- `^sys > *` ≡ `depth(-1) > *`

Attribute filters remain unchanged and MAY be combined with depth predicates: `[key]`, `[type]`, `[tag]`, etc. Overlap note: `[key="K"]` MAY return multiple matches; use depth ordering or additional filters (e.g., `:depth(>0):first`) to choose the newest instance.

### 3.5 Attributes
Selectors MUST support attributes:  
`[offset] [ttl] [priority] [cycle] [created_at_ns] [created_at_iso] [nodeType] [id] [role] [kind]`

#### 3.5.1 Grouped Shorthand (Normative)
In addition to `[attr]` filters, implementations MUST support a grouped shorthand form using parentheses immediately after a type token. Example:
```
.cb(kind='important' ttl=5)
```
is semantically identical to:
```
.cb[kind='important'][ttl=5]
```
Attributes inside `(...)` are separated by space or comma. This is purely syntactic sugar; engines MUST normalize grouped forms to canonical bracket filters before evaluation. Mixing forms is allowed: `.cb(kind='summary')[ttl<=2]`.

Parentheses immediately following a type token are reserved for this shorthand and MUST NOT collide with pseudos; pseudo-classes continue to use the `:` prefix (e.g., `:depth(...)`).

#### 3.5.2 Key Selection Semantics (Normative)
- `[key="K"]` filters by logical family handle. Multiple components MAY share the same `key` within a snapshot; results MAY include multiple instances.
- Default result ordering for same‑`key` sets MUST follow 02 – Invariants §3.6 (Determinism): primary `depth` DESC, then `created_at_ns` DESC, then `creation_index` DESC, then `id` ASC.
- Implementations MUST support stable windowing over selector results (skip/limit) to enable newest/older selection. Engines MAY expose this via API chaining (e.g., `.limit(1)`, `.skip(1).limit(1)`) or equivalent selector options.

Comparison operators: `=`, `!=`, `<`, `<=`, `>`, `>=`.
 
Implementations create attributes via node headers defined in [02 – Invariants §3] (see §3.5 “Attribute Creation and Stability”). Missing optional attributes compare as `null` per §5.5; mandatory headers MUST be populated.

#### 3.5.3 Derived Facets (Snapshot‑only)

The following derived facets are defined only on snapshots (turns), not on the working state:

- `born_turn` — index of the turn where this projection was created.  
- `age` — number of turns since `born_turn` at the selected snapshot.  
- `depth` — structural index of the containing segment in `^seq` at the selected snapshot.  
- `sdepth` (optional) — structural nesting depth from root.  

Validators MUST reject selectors that reference these facets when no `@t` is present.

Examples:

```
@t0 (d1, *:cb[kind="summary"]) [age<2]
@t0..@t-3 (d2, *:cb) @mode=latest [depth=2]
```

### 3.6 Combinators
- Descendant: `A B`  
- Child: `A > B`

### 3.7 Snapshots
Snapshots are addressable with `@t0` (latest committed turn/snapshot), `@t-1` (older), `@cN` (absolute cycle), or `@*` (all).
If no `@t` is given, the selector applies to the working state (the live Active Head). Implementations MUST NOT silently assume `@t0`.

Range addressing across snapshots MUST be supported with **inclusive** semantics:
- `@t-<start>..<end>` — inclusive range using double-dots  
- `@t-<start>:<end>` — inclusive range using a colon (**interchangeable** with `..`)

**Contract**
- Presence of a SnapshotRange ⇒ return **RangeDiffLatestResult**.  
- Otherwise ⇒ return **ordered list of IDs**.

Ranges, `@mode`, and time‑derived facets (see §3.5.3) are valid only on snapshots (i.e., when a `@t` is present). Validators MUST reject such facets when no `@t` is provided.

### 3.8 Position

Hops may include:

- `dN` or `depth(N)` to navigate into `^seq` (structural depth).  
- `seg` to indicate a segment container explicitly.  
- `mc`, `cb`, and facet filters as before.  

Examples:

```
(d0, mc[offset=0], *:cb[role="user"])
(d1, seg, mc[offset=0], *:cb[role="assistant"])
(d2, seg, *:cb[kind="summary"])
(d-1, *:cb[role="system"])
```

### 3.9 Insertion

Use structural cues inside the selector, not API flags:

- Trailing comma `,` = insertion point.  
- `end` = append at the end of children.  
- `+N` = insert after child N.  
- `-N` = insert before child N.  

Examples:

```
(d0, seg, mc[offset=0], )                 # insert at start
(d0, seg, mc[offset=0], end)              # insert at end
(d0, seg, mc[offset=0], +1)               # insert after child 1
```

### 3.7.1 Conformance for Range Results (Normative)

- When a SnapshotRange is present in a selector, the result type MUST be `RangeDiffLatestResult` by default.
- No migration or legacy result modes are defined pre‑v1.

**Examples**
```text
# Non-range (IDs)
ctx.select("@t0 ^seq .mt:depth(1) .mc > .cb[role='assistant']")
→ ["cb:a1", "cb:a7", ...]

# Range (RangeDiffLatestResult)
ctx.select("@t-3..@t0 ^seq .mt .cb[nodeType='cb'][kind='summary']")
→ {
     "query": "@t-3..@t0 ^seq .mt .cb[nodeType='cb'][kind='summary']",
     "snapshots": [ { "label":"@t0" }, { "label":"@t-1" }, { "label":"@t-2" }, { "label":"@t-3" } ],
     "diffs": [
       { "from":{"label":"@t0"}, "to":{"label":"@t-1"}, "added_ids":[], "removed_ids":["cb:sum:c101"], "changed":[] },
       { "from":{"label":"@t-1"}, "to":{"label":"@t-2"}, "added_ids":["cb:sum:c102"], "removed_ids":[], "changed":[] },
       { "from":{"label":"@t-2"}, "to":{"label":"@t-3"}, "added_ids":[], "removed_ids":[], "changed":[] }
     ],
     "mode": "pairwise"
   }
```

---

## 4. Grammar (EBNF)
```ebnf
# SnapshotRange presence affects ONLY the return type (IDs vs RangeDiffLatestResult).
Selector   ::= [Snapshot] Group { "," Group }
Group      ::= Chain
Chain      ::= Step { Combinator Step }
Step       ::= Simple
Combinator ::= " " | ">"
Simple     ::= [Root] [ID] [Type] [AttrGroup?] [Attr*] [Pseudo*] | "*"

Snapshot        ::= SnapshotAtom | SnapshotRange
SnapshotAtom    ::= "@" ( "t" [ "-" ] Digit+ | "c" Digit+ | "*" )
SnapshotRange   ::= SnapshotAtom ".." SnapshotAtom   # inclusive; ":" interchangeable with ".."
Root            ::= "^" ("sys" | "seq" | "ah" | "root")
ID              ::= "#" Identifier
Type            ::= "." Identifier
Attr            ::= "[" Key [Op Value] "]"
AttrGroup       ::= "(" AttrPair { (" " | ",") AttrPair } ")"
AttrPair        ::= Key [Op Value]
Pseudo          ::= ":" Identifier [ "(" ValueList ")" ]

ValueList  ::= Value { "," Value }
Value      ::= Number | String | Identifier | Range
Range      ::= Number "-" Number
Number     ::= [ "-" ] Digit+ [ "." Digit+ ]
String     ::= '"' StringChar* '"' | "'" StringChar* "'"
Identifier ::= Letter ( Letter | Digit | "_" | "-" | ":" )*
Key        ::= Identifier
Op         ::= "=" | "!=" | "<" | "<=" | ">" | ">="
Letter     ::= "a".."z" | "A".."Z"
Digit      ::= "0".."9"
StringChar ::= [^"'] | "\\" ( '"' | "'" | "\\" )
```

**Normalization Rule (Normative)**  
- Every `AttrGroup` MUST be expanded into the equivalent sequence of `Attr` before evaluation.  
- Example: `.cb(role='system' ttl=20)` ⇒ `.cb[role='system'][ttl=20]`.  
- Empty groups `()` are invalid and MUST raise an error.

Constraints:
- Endpoints MUST be the same kind: "@t..@t" or "@c..@c".
- "@*" MUST NOT appear inside a range.
- Range is inclusive of both endpoints.

### Range Select Output — RangeDiffLatestResult (Normative)

When a SnapshotRange is present, `ctx.select` returns:

RangeDiffLatestResult {
  "query": string,                      // original selector
  "snapshots": SnapshotRef[],           // expanded & resolved, sorted DESC by cycle (newest→oldest)
  "diffs": DiffBlock[],                 // pairwise steps: (s0→s1), (s1→s2), ...
  "mode": "pairwise",                   // reserved for future (e.g., "nodewise","baseline")
  "limits"?: { "maxSnapshots"?:number, "maxChangesPerSnapshot"?:number, "truncated"?:boolean },
  "warnings"?: string[],
  "errors"?: Array<{ "code":string, "message":string, "fatal":boolean }>
}

# Depth forms (UD)
ctx.select("^seq .mt:depth(>0)")
ctx.select(".mt:depth(0)")                # Active Head
ctx.select(".mt:depth(<0)")               # System depth
ctx.select(".mt:depth({1,2}) .cb")        # Newest two historical turns

SnapshotRef { "kind":"t"|"c", "value":number, "label":string, "cycle":number }

DiffBlock {
  "from": SnapshotRef,                  // newer snapshot
  "to":   SnapshotRef,                  // older snapshot
  "added_ids":   string[],              // present in 'from' but not in 'to'
  "removed_ids": string[],              // present in 'to'   but not in 'from'
  "changed": Array<{
    "id": string,
    "fields": string[],                 // headers that differ (e.g., ["ttl","content_hash","parent","offset"])
    "delta"?: Record<string,{ "from":any, "to":any }>,
    "content_diff"?: {                  // present iff options.includeContent=true AND "content_hash" changed
      "mode":"unified"|"jsonpatch"|"tokens",
      "mime"?: string,
      "summary": { "bytesNewer":number, "bytesOlder":number, "identical":boolean, "truncated":boolean },
      "diff"?: string,
      "patch"?: any,
      "hunks"?: any[]
    }
  }>,
  "stats"?: { "added":number, "removed":number, "changed":number }
}

Changed detection (headers)
- A node is “changed” iff the same id is present in both snapshots of the pair and any tracked header differs
  (e.g., ttl, priority, parent, offset, nodeType, role, kind, content_hash, created_at_ns, creation_index).
  delta carries scalar old/new values. If content_hash changed and options.includeContent=true,
  implementations SHOULD attach a content_diff.

Ordering and determinism
- snapshots sorted newest→oldest (DESC by cycle).
- Within each DiffBlock, added_ids/removed_ids SHOULD be ordered by the canonical sibling order of the from snapshot,
  then by id.
- Deterministic: identical inputs produce identical outputs byte-for-byte.

### ctx.select Options (applies when a SnapshotRange is present)

Signature:
  ctx.select(selector, options?)

Options:
- includeContent?: boolean
  When true, and a node’s content_hash is in changed.fields, attach a content_diff to that changed item.

- contentMode?: "unified"|"jsonpatch"|"tokens"
  Default "unified". Controls the content_diff representation.

- contextLines?: number
  Unified diff context lines (default 3).

- materialize?: ("content"|"mime"|"encoding"|"content_hash")[]
  When present, servers MAY inline minimal fields needed to compute or validate diffs.

- maxSnapshots?: number
  Cap range expansion.

- maxChangesPerSnapshot?: number
  Cap size of each DiffBlock; set limits.truncated=true on truncation.

Non-range calls ignore these options and continue to return IDs.

---

## 5. Attribute Comparison Rules

### 5.1 Data Type Detection
Attribute values MUST be compared using appropriate data types:

- **Numeric attributes**: `offset`, `ttl`, `priority`, `cycle`, `created_at_ns`
- **String attributes**: `nodeType`, `id`, `role`, `kind`, `created_at_iso`

### 5.2 Numeric Comparison
For numeric attributes, implementations MUST:
1. Parse attribute values as numbers (integers or floats)
2. Apply numeric comparison semantics for `<`, `<=`, `>`, `>=`
3. Treat `None` or `null` values as distinct from numeric values
4. Use exact equality for `=` and `!=`

```
[ttl>5]      # numeric: ttl greater than 5
[offset<0]   # numeric: offset less than zero  
[cycle=42]   # numeric: exact equality
```

### 5.3 String Comparison  
For string attributes, implementations MUST:
1. Use case-sensitive string comparison by default
2. Apply lexicographic ordering for `<`, `<=`, `>`, `>=`
3. Support exact string matching for `=` and `!=`
4. Treat empty strings as distinct from null/undefined

```
[role='user']           # exact string match (case-sensitive)
[nodeType='cb'][kind='summary'] # exact match by canonical nodeType and kind
[id!='abc123']          # string inequality
```

### 5.4 Mixed Type Handling
When attribute type is ambiguous:
1. Attempt numeric parsing first for comparison operators `<`, `<=`, `>`, `>=`
2. Fall back to string comparison if numeric parsing fails
3. For `=` and `!=`, use type-preserving comparison
4. Implementations SHOULD document their type coercion rules

### 5.5 Special Values
- **`None`/`null`**: Only matches with `=` or `!=`, never with ordering operators
- **Empty string `""`**: Treated as valid string, distinct from `None`
- **Boolean**: Convert to string ("true"/"false") for comparison

---

## 6. Examples

# all pre-context content blocks in active head

ctx.select("@t0 ^ah :pre .cb")

# assistant replies from last sealed turn
ctx.select("^seq .mt:depth(1) .mc > .cb[role='assistant']")

# expired or soon-to-expire blocks
ctx.select("^root .cb[ttl<=1]")

# a specific node by ID
ctx.select("#abc123def")

# last three historical turns, all user messages
ctx.select("^seq .mt:depth(1,2,3) .cb[role='user']")

# last three historical turns (range form), all user messages
ctx.select("^seq .mt:depth(1-3) .cb[role='user']")

# last five historical turns (range form)
ctx.select("^seq .mt:depth(1-5)")

# select across snapshot range t-5 through t-1 (inclusive)
ctx.select("@t-5:@t-1 ^ah .cb")

# find a specific node across all available snapshots
ctx.select("@* #abc123def")

# position node N under a specific historical turn (depth targeting)
# select the target turn at depth 3 and then place N as post-context via offset>0
# (actual placement is done by the implementation API; selector here shows targeting)
ctx.select("^seq .mt:depth(3)")

# newest instance for a logical key K (sealed history)
ctx.select("^seq .mt:depth(1) .cb[key='K']")

# key selection with windowing (normative semantics)
# engines MUST support equivalent of newest and second-newest selection
select("[key='K']").limit(1)                    # newest instance (per deterministic order)
select("[key='K']").skip(1).limit(1)            # second-newest

# composed examples
select("[key='K']").where(".mt:depth(>0)")     # all history overlaps for K (excluding AH)
select(".mt:depth(<0)[key='policy-banner']")    # system-scoped by key

# shorthand group form (equivalent to multiple [..] filters)
ctx.select("^seq .mt:depth(1) .cb(role='assistant' ttl<=2)")
# equivalent to:
ctx.select("^seq .mt:depth(1) .cb[role='assistant'][ttl<=2]")

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
  "spec_version": "PACT/0.1.0",
  "root": {
    "children": [
      { "id": "sys-1", "nodeType": "^sys", "children": [
        { "id": "cb:sysA", "nodeType": "cb", "role": "system", "kind": "text", "offset": 0, "content": "S" }
      ]},
      { "id": "seq-1", "nodeType": "^seq", "children": [
        { "id": "mt:1", "nodeType": "mt", "children": [
          { "id": "cb:u1", "nodeType": "cb", "role": "user", "kind": "text", "offset": 0, "ttl": 2, "content": "U1" }
        ]},
        { "id": "mt:2", "nodeType": "mt", "children": [
          { "id": "cb:a1", "nodeType": "cb", "role": "assistant", "kind": "text", "offset": 0, "ttl": 1, "content": "A1" }
        ]}
      ]},
      { "id": "ah-1", "nodeType": "^ah", "children": [
        { "id": "cb:u2", "nodeType": "cb", "role": "user", "kind": "text", "offset": 0, "content": "U2" }
      ]}
    ]
  }
}
```

Implementations MUST produce exactly these results (IDs and order):

1. `@t0 ^sys .cb` → `["cb:sysA"]`
2. `@t0 ^seq .mt:depth(1)` → `["mt:2"]`
3. `@t0 ^seq .mt:depth(1,2)` → `["mt:1","mt:2"]` (newer→older)
4. `@t0 ^seq .mt:depth(1-2) .mc > .cb` → `["cb:u1","cb:a1"]`
5. `@t0 #cb:u2` → `["cb:u2"]`

## 6.3 Additional Range Fixture (Normative)

To validate range depth selectors, the following minimal snapshot fixture at `@t0` MUST produce the listed results:

```json
{
  "spec_version": "PACT/0.1.0",
  "root": {
    "children": [
      { "id": "seq-2", "nodeType": "^seq", "children": [
        { "id": "mt:1", "nodeType": "mt", "children": [ { "id": "cb:u1", "nodeType": "cb", "role": "user", "kind": "text", "offset": 0, "content": "U1" } ] },
        { "id": "mt:2", "nodeType": "mt", "children": [ { "id": "cb:u2", "nodeType": "cb", "role": "user", "kind": "text", "offset": 0, "content": "U2" } ] },
        { "id": "mt:3", "nodeType": "mt", "children": [ { "id": "cb:u3", "nodeType": "cb", "role": "user", "kind": "text", "offset": 0, "content": "U3" } ] }
      ]}
    ]
  }
}
```

- Query: `@t0 ^seq .mt:depth(1-3) .cb[role='user']`
  Expect: `["cb:u1","cb:u2","cb:u3"]` (newer→older across matched turns)


## 7. Conformance Checklist

An implementation is conformant if:

1. Roots ^sys, ^seq, ^ah, ^root MUST be supported.
2. Types .mt, .mc, .cb MUST be supported.
3. User-assigned nodeType MUST be matchable via namespaced type tokens (e.g., `.cb:summary`) and via attributes (e.g., `[nodeType='cb:summary']`).
4. IDs MUST be matched exactly (case-sensitive).
5. All specified pseudos MUST be implemented.
6. Attributes MUST support all comparison operators.
7. Combinators (descendant, child) MUST be supported.
8. Snapshots (@t0, @t-1, @cN, @*) MUST be supported.
9. Results MUST be ordered canonically.
10. Queries MUST be pure (no side effects).
11. Invalid selectors MUST raise errors.
12. `:depth(n1,n2,...)` comma-separated lists MUST be supported.
13. Numeric vs string attribute comparison rules MUST be followed.
14. Data type detection for attributes MUST be implemented correctly.
15. Default scope if omitted MUST be the working state (Active Head) (see §3.7); implementations MUST NOT assume `@t0` for queries without an explicit snapshot.
16. Snapshot range addressing (`@tA..B` and `@tA:B`) MUST be supported with inclusive semantics and both operators MUST be interchangeable.
17. Grouped-attribute shorthand `Type(attr1 op val1 [ ,|\s attr2 op val2 ... ])` MUST be accepted and normalized to canonical bracket filters prior to evaluation; `[key=val]` syntax remains fully valid.

- MUST parse SnapshotRange and reject mixed kinds ("@t..@c") or wildcard endpoints ("@*" in ranges).
- MUST return RangeDiffLatestResult when a SnapshotRange is present in the selector.
- MUST return a list of IDs when the selector targets a single snapshot or "@*".
- MUST sort snapshots newest→oldest and compute pairwise diffs in that order.
- MUST be deterministic (stable ordering and byte-identical outputs for identical inputs).
- SHOULD expose limits and set truncated=true when caps are hit.

### 7.1 Golden Tests (Normative)

Given the minimal snapshot fixture in §6.2, implementations MUST produce exactly these ordered ID results:

- Query: `@t0 ^sys .cb`
  Expect: `["cb:sysA"]`

- Query: `@t0 ^seq .mt:depth(1)`
  Expect: `["mt:2"]`

- Query: `@t0 ^seq .mt:depth(1,2)`
  Expect: `["mt:1","mt:2"]` (newer→older)

- Query: `@t0 ^seq .mt:depth(1-2) .mc > .cb`
  Expect: `["cb:u1","cb:a1"]`

- Query: `@t0 ^seq .mt:depth(1) > .cb`
  Expect: `["cb:a1"]`

- Query: `@t0 #cb:u2`
  Expect: `["cb:u2"]`

- Query: `@t0 .cb[role='assistant']`
  Expect: `["cb:a1"]`

- Query: `@t0 ^seq .mt:depth(1-2) .cb[ttl<=1]`
  Expect: `["cb:a1"]`

- Query: `@t0 ^seq .mt:depth()`
  Expect: MUST raise an error (invalid selector: empty depth)

- Query: `@t0 ^seq .mt:depth(3) .cb[role='user']`
  Expect: `[]` (empty match, not an error)

[← 03-lifecycle-ttl](03-lifecycle-ttl.md) | [↑ Spec Index](../README.md) | [→ 05-snapshots](05-snapshots.md)

