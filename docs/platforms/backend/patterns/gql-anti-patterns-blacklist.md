# GQL Anti-Patterns Blacklist (Ejtmaa)

Any listed anti-pattern is blocked unless explicitly approved with documented exception.

## Security Anti-Patterns

1. **Resolver-only authorization for scoped data**
   - Missing guard in `willPrepare`/`getRootOrmParent`.
2. **Implicit principal assumptions**
   - Using principal-dependent query path without null checks.
3. **Bypassing bridge security contract**
   - Direct resolver ORM calls for bridge-owned entity queries.
4. **Duplicated principal guard paths**
   - Repeating the same principal check in entity `willPrepare` while role base `getRootOrmParent` already enforces it for the same parent mode.

## Performance Anti-Patterns

1. **Unbounded root-many retrieval**
   - no max clamp and no deterministic order.
2. **Depth explosion**
   - `maxLevel` and `deepInclude` guard changes require an explicit replacement guard.
3. **List extras over-computation**
   - expensive extras calculated on many paths by default.

## Mapping Anti-Patterns

1. **Filter/sort on non-loaded attrs**
   - missing `requiredOrmAttrs` or include attrs.
2. **Union discriminator missing**
   - polymorphic routing without guaranteed discriminator loading.
3. **Bridge args mismatch**
   - resolver parent payload shape not aligned with bridge parent type.
4. **User text inside `sequelize.literal`**
   - dynamic SQL built by interpolating request text.
5. **Manual re-declaration of ORM-native relations**
   - adding `static relations` entries for relations that already exist in model associations.

## Contract Anti-Patterns

1. **Bridge not registered**
   - query uses bridge not in `registeredBridges`.
2. **Cross-role schema/bridge ownership mismatch**
   - `schemaIdent` mismatch or bridge registered under wrong schema.
3. **Context contract drift**
   - missing required context keys (`lang`, list controls, `by`, principal).
4. **Forced STATIC root-one override despite owner-parent base support**
   - overriding `getRootOrmParent(..., "one")` to `"STATIC"` for `{me: true, id}` flows when role base already provides owner-parent resolution.

## Bridge Base Anti-Patterns

1. **Missing `loadAttr` override in role bridge base**
   - Role bridge base without MultiLang/enum resolution causes JSONB fields to return raw objects, aborting entire queries.
2. **Missing `trans`/`toEnum`/`toManyEnums` in role bridge base**
   - Entity bridges depend on these for field-specific translations; absence forces ad-hoc patching in entity bridges.
3. **Missing `willPrepare` base in role bridge base**
   - Entity bridges may override `willPrepare`; missing base method breaks the override chain.

## Style Drift Anti-Patterns

1. **Importing incompatible project conventions**
   - naming/typing/flag strategy not aligned with Ejtmaa baseline.
2. **Hardcoded identities in production paths**
   - static `findByPk(...)` style reused in runtime auth path.
3. **Inconsistent flag semantics**
   - introducing alternate equivalents (`self` vs `me`) without compatibility plan.
4. **Untyped role resolvers**
   - keeping `resolvers(): any` after generated resolver types are available.
5. **Filter enum contract drift**
   - role filter inputs use object enums where the contract requires `*_Value` wrappers.
6. **Unrequested lazy relation mapping**
   - introducing `lazy` relation tuples without explicit requirement for non-ORM derived relation paths.
7. **Root-one parent strategy drift**
   - mixing owner-parent and forced static strategies in the same role surface without an explicit documented exception.
