---
name: website-route-static-before-parametric
description: Prevents website first-match route collisions by registering fixed URL segments before parametric routes on the same prefix in routes.ts. Use when adding routes under website/, debugging wrong page loads, bigint cast errors on slug params, or auditing routes.ts registration order.
---

# Website Route Static-Before-Parametric

## When to Use

Use when:

- adding a new route whose path shares a prefix with an existing `:id` (or `:formType`) route,
- reordering entries in `website/src/resources/configs/routes.ts`,
- debugging a customer URL that opens the wrong screen (form vs detail),
- seeing backend errors like `invalid input syntax for type bigint: "new"`,
- reviewing a PR that introduces `/prefix/:id` near `/prefix/<fixed-segment>`.

## Instructions

1. **Confirm first-match semantics.** `web-core` walks `routes` in object key order; the first path pattern that matches wins.
2. **Inventory the prefix family.** For each shared prefix, list fixed-segment routes (`new`, `review`, `return`) and parametric routes (`:id`).
3. **Reorder `const routes` only.** Move every fixed route above the parametric sibling in the customer section block.
4. **Verify against** `route-registry-contract.md` section 4 and section 5 path tables.
5. **Smoke-check** fixed URLs and numeric-id detail URLs after reorder.
6. **Update docs** when paths change; cite invariant W41.

## Diagnosis workflow

```
Wrong screen or bigint slug error on a customer URL
  -> read URL path; note the segment after the prefix
  -> grep routes.ts for that prefix
  -> if :id route appears BEFORE the fixed route -> reorder (fixed first)
  -> retest URL
```

## Examples

**Bad** -- `/customer/feature/new` matches detail:

```ts
CustomerDetailRoute: { path: customerRouter("/feature/:id"), ... },
CustomerFormRoute: { path: customerRouter("/feature/new"), ... },
```

**Good** -- fixed segments first:

```ts
CustomerFormRoute: { path: customerRouter("/feature/new"), ... },
CustomerDetailRoute: { path: customerRouter("/feature/:id"), ... },
```

## Reference

- Rule: `.cursor/rules/website-route-static-before-parametric.mdc`
- Registry governance: `.cursor/rules/website-route-registry-governance.mdc`
- Contract: `docs/platforms/website/route-registry-contract.md`
- Invariant: `docs/invariants/website.md` W41
