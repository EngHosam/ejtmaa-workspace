# GQL Union Routing Reference (BridgeBase)

## Purpose

This reference defines the canonical bridge-first pattern for GraphQL union fields in Ejtmaa.

Use it when one GraphQL field should expose multiple model-backed relation branches chosen by a discriminator.

## When To Use Union

Use a GraphQL union field when:
- one semantic field can represent different object types,
- branch choice is deterministic from persisted discriminator attrs,
- branch relations already exist as concrete ORM relations.

Do not use union when:
- the branch discriminator is missing or unstable,
- a fixed single relation is sufficient,
- branch output should remain explicit independent fields by contract.

## Required Contract

1. SDL defines union members explicitly.
2. Bridge declares `static unions` with:
   - union field ident,
   - concrete `gqlIncludeIdents`,
   - `requiredOrmAttrs` for discriminator fields.
3. Bridge implements deterministic `getUnionGQLIncludeIdent(...)`.
4. Schema resolvers stay root-only; no manual union-member field resolvers.
5. ORM keeps concrete relations; do not invent fake union ORM relation.

## Runtime Mechanics (BridgeBase)

The runtime flow is implemented by framework behavior in:
- `backend/eng-hosam/@nodejs/gql/src/BridgeBase.ts`

Sequence:
1. `static unions` is declared in entity bridge.
2. Bridge boot builds internal mapping from concrete include idents to union field.
3. `prepareRelationsBridges()` prepares member bridges as fragment sub-bridges when ask is on the union field.
4. `loadRelations()` calls `getUnionGQLIncludeIdent(...)` to choose active member per row.
5. Chosen member is attached under the union field key.
6. Member bridge output contains `__typename` from `prepareOneGQLModel(...)`.
7. GraphQL resolves the union member from that `__typename`.

## SDL And Bridge Template

SDL:

```graphql
union _ExampleTarget = _TypeA | _TypeB

type _Example {
    id: ID!
    target_type: _ExampleTargetType
    target: _ExampleTarget
}
```

Bridge:

```ts
static unions: Unions = {
    target: {
        gqlIncludeIdents: ["typeA", "typeB"],
        requiredOrmAttrs: ["target_type"]
    }
};

getUnionGQLIncludeIdent(gqlUnionIdent, model) {
    if (gqlUnionIdent === "target") {
        switch (model.get("target_type")) {
            case "TYPE_A":
                return "typeA";
            case "TYPE_B":
                return "typeB";
        }
    }
}
```

## Anti-Patterns (Forbidden)

- Adding schema-level manual resolvers for union member fields in model-backed paths.
- Declaring a fake ORM relation whose only purpose is matching union field name.
- Routing union branches without loading discriminator attrs.
- Non-deterministic routing (branch choice depends on request order or side effects).
- Mixing old explicit relation fields and union field without documented compatibility reason.

## Verification Checklist

- [ ] mixed-discriminator dataset returns expected union member type per row
- [ ] no cross-branch leakage (`TypeA` row never resolves as `TypeB`)
- [ ] schema root resolvers remain thin bridge-entry only
- [ ] discriminator attrs are present in bridge required attr strategy
- [ ] codegen union types are generated as expected
- [ ] website GraphQL mirror is synced

## Worked Examples

### Ejtmaa: Supervisor customer reads

- `backend/src/app/gql/bridges/supervisor/CustomerBridge.ts`
- `backend/src/app/gql/definitions/supervisor.graphql`

Pattern:

- root `many`: `customers(filter: _CustomerFilter)`
- root `one`: `customer(id: ID!)`
- filter mapping in bridge `getOrmFindOptions`
