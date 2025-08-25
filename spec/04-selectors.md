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
- An **ordered list of node IDs**.  
- Nodes are returned in canonical sibling order (see [02 – Invariants](02-invariants.md) §4.3).

---

## 3. Syntax Elements
### 3.1 Roots
- `^sys` — System Header  
- `^seq` — Sealed sequence of turns  
- `^ah` — Active Head (in-progress turn)  
- `^root` — Root container

### 3.2 Types
- `.mt` — MessageTurn  
- `.mc` — MessageContainer  
- `.cb` — ContentBlock  
- Namespaced user-assigned types MUST also be supported. A type token like `.cb:summary` MUST match nodes whose `nodeType` equals `"cb:summary"`.
- Equivalence (Normative): `.cb:summary` and `[nodeType='cb:summary']` MUST be treated as equivalent forms and MUST return identical results.
- `kind` is not a type; query it via attributes (e.g., `.cb[kind='tool_call']`). Bare `.tool_call` is not standardized and MUST NOT be required for conformance.

### 3.3 IDs
- `#<id>` — select by node ID (opaque string)

### 3.4 Pseudo-classes
- `:pre` — nodes with `offset < 0`  
- `:core` — nodes with `offset = 0`  
- `:post` — nodes with `offset > 0`  
- `:depth(n)` — select a turn at depth `n` in `^seq` (newest = 1)  
- `:depth(n1,n2,...)` — select turns at multiple depths (comma-separated list)  
- `:depth(n1-n2)` — select turns within inclusive depth range `n1` through `n2` (newest = 1)  
- `:first` — first sibling among a set  
- `:last` — last sibling among a set  
- `:nth(n)` — nth sibling (1-based index)

Offset-based pseudos map to canonical offset semantics defined in [02 – Invariants §4.1–4.2](02-invariants.md).

### 3.5 Attributes
Selectors MUST support attributes:  
`[offset] [ttl] [priority] [cycle] [created_at_ns] [created_at_iso] [nodeType] [id] [role] [kind]`

Comparison operators: `=`, `!=`, `<`, `<=`, `>`, `>=`.
 
Implementations create attributes via node headers defined in [02 – Invariants §3] (see §3.5 “Attribute Creation and Stability”). Missing optional attributes compare as `null` per §5.5; mandatory headers MUST be populated.

### 3.6 Combinators
- Descendant: `A B`  
- Child: `A > B`

### 3.7 Snapshots
Snapshots are addressable with `@t0` (current), `@t-1` (previous), `@cN` (absolute cycle), or `@*` (all).
If omitted, the snapshot reference MUST default to `@t0`.
Implementations MUST NOT require explicit `@t0` for queries against the current snapshot.

---

## 4. Grammar (EBNF)
```ebnf
Selector   ::= [Snapshot] Group { "," Group }
Group      ::= Chain
Chain      ::= Step { Combinator Step }
Step       ::= Simple
Combinator ::= " " | ">"
Simple     ::= [Root] [ID] [Type] [Attr*] [Pseudo*] | "*"

Snapshot   ::= "@" ("t" [ "-" ] Digit+ | "c" Digit+ | "*" )
Root       ::= "^" ("sys" | "seq" | "ah" | "root")
ID         ::= "#" Identifier
Type       ::= "." Identifier
Attr       ::= "[" Key [Op Value] "]"
Pseudo     ::= ":" Identifier [ "(" ValueList ")" ]

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
[nodeType='cb:summary'] # exact match with namespace
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

# last three sealed turns, all user messages
ctx.select("^seq .mt:depth(1,2,3) .cb[role='user']")

# last three sealed turns (range form), all user messages
ctx.select("^seq .mt:depth(1-3) .cb[role='user']")

# find a specific node across all available snapshots
ctx.select("@* #abc123def")

# position node N under a specific sealed turn (depth targeting)
# select the target turn at depth 3 and then place N as post-context via offset>0
# (actual placement is done by the implementation API; selector here shows targeting)
ctx.select("^seq .mt:depth(3)")

---

## 6.2 Canonical Selector Tests (Normative)

Given this minimal snapshot fixture at `@t0`:

```json
{
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
3. `@t0 ^seq .mt:depth(1,2)` → `["mt:1","mt:2"]` (older→newer)
4. `@t0 ^seq .mt:depth(1-2) .mc > .cb` → `["cb:u1","cb:a1"]`
5. `@t0 #cb:u2` → `["cb:u2"]`

## 6.3 Additional Range Fixture (Normative)

To validate range depth selectors, the following minimal snapshot fixture at `@t0` MUST produce the listed results:

```json
{
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
  Expect: `["cb:u1","cb:u2","cb:u3"]` (older→newer across matched turns)


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
15. Default snapshot if omitted MUST be `@t0` (see §3.7); implementations MUST NOT require explicit `@t0` for current snapshot queries.

### 7.1 Golden Tests (Normative)

Given the minimal snapshot fixture in §6.2, implementations MUST produce exactly these ordered ID results:

- Query: `@t0 ^sys .cb`
  Expect: `["cb:sysA"]`

- Query: `@t0 ^seq .mt:depth(1)`
  Expect: `["mt:2"]`

- Query: `@t0 ^seq .mt:depth(1,2)`
  Expect: `["mt:1","mt:2"]` (older→newer)

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

