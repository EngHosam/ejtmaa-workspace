# GQL Evaluation Suite

Use this suite to validate any schema/bridge output before acceptance.

## Suite A: Contract Integrity

1. Schema context includes principal + lang + list controls + `by`.
2. Bridges are registered in owning schema.
3. Bridge ownership matches `schemaIdent`.
4. Resolver delegation follows bridge-root pattern for model-backed reads.

## Suite B: Security Integrity

1. Scoped queries enforce principal checks.
2. Unauthorized contexts fail deterministically.
3. No resolver-only security for bridge-owned scoped paths.

## Suite C: Query Integrity

1. Root-many has bounded length.
2. Root-many has deterministic order.
3. Root-one path has deterministic id mapping.
4. Pagination mode behavior (`pointer/page/upOnly`) is coherent.

## Suite D: Mapping Integrity

1. Every sort/filter/extra/union attr has ORM load path.
2. Required attrs are declared for bridge logic.
3. Union discriminator attrs are guaranteed selected.

## Suite E: Performance and Depth

1. `maxLevel` finite and active.
2. `deepInclude` strategy active.
3. list extras behavior is guarded (`ignoreExtra` policy).
4. computed registered attrs are justified.

## Suite F: Compatibility

1. Output follows `gql-schema-bridge-authoring-standard.md`.
2. Output does not violate blacklist.
3. Refusal matrix conditions were evaluated.
4. Pattern choice mapped to `gql-pattern-atlas.md`.

## Suite G: Framework Integrity

1. Provider/schema construction contract is preserved.
2. Bridge registration/ownership invariants remain valid.
3. Asks/include/attrs behavior remains compatible with framework expectations.
4. Any temporary drift (`resolvers(): any`, direct resolver ORM fallback) is explicitly documented.

## Execution Result Format

```text
Evaluation Result:
- Contract Integrity: pass/fail
- Security Integrity: pass/fail
- Query Integrity: pass/fail
- Mapping Integrity: pass/fail
- Performance and Depth: pass/fail
- Compatibility: pass/fail
- Framework Integrity: pass/fail
Final: ACCEPT / REVISE / REJECT
```
