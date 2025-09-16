# Migration drafts (archival, non-normative)

This file preserves the removed migration recipe content from `spec/04-queries.md` for historical reference. This content is not part of the normative specification.

## 04 – Context Queries — Migration Recipe (archived)

### Regex replace (git-run friendly examples)

```text
s/\.block:([A-Za-z0-9_]+)/.\1/g   # `.block:summary` → `.summary`
s/cont\[offset=([0-9-]+)\]/\1/g    # `cont[offset=0]` → `0`
s/c@([0-9]+)/ (\1)/g                # best-effort cleanup for c@→ coordinate form (manual verify)
s/\*\@[\+\-]/[use-offset-filter]/g # remove \*@+/- occurrences; replace with offset filters
```

### Linter rules (archived auto-fix guidance)

- Flag any `cont[...]`, `c@`, `.block:...`, `*@+/-` uses as errors.
- Auto-fix `.block:TYPE` → `.TYPE`.
- Auto-fix `cont[offset=N]` → `N` inside coordinates/paths.

Add note: run migration branch + tests; update examples and unit tests.
