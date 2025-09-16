# Adopter extensions and historical examples (non-normative)

This annex collects non-normative adopter guidance and historical examples that were removed from the main PACT specification, including uses of the `kind` attribute.

---

## Extracted: "kind":"summary",

From spec/09-annex-conformance.md (Example JSON node):

```json
{
"id":"node-abc",
"nodeType":"cb",
"parent_id":"mc-17",
"offset":0,
"kind":"summary",
"created_at_iso":"2025-09-16T08:00:00Z",
"ttl":3
}
```

## Extracted: Non-normative: `kind` is an implementation convenie…

From spec/01-architecture.md:

Non-normative: `kind`
- `kind` is an implementation convenience and is explicitly NON-NORMATIVE. Runtimes MAY use a `kind` attribute to categorize nodes (for example `kind='summary'`) and MAY expose it via property-search queries such as `{ kind="summary" }`.
- PACT conformance MUST NOT depend on specific `kind` values. Consumers should not assume consistent semantics across implementations unless an adopter-level extension contract is published.

---

## Extracted: Legacy tokens to flag … `:kind` pseudo-syntax

From spec/04-queries.md (Linter rules):

- Flag any usage of legacy tokens as **errors**; do NOT auto-fix in v1. Legacy tokens to flag: `cont[...]`, `c@`, `.block:...`, `*@+/-`, `.mc`, `:kind` pseudo-syntax.
