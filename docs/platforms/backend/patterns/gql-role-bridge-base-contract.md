# GQL Role Bridge Base Contract (Ejtmaa)

## Purpose

Every role bridge base (`CustomerBridgeBase`, `SupervisorBridgeBase`) must implement the standard attribute loading, translation, and enum resolution contract.

Without `loadAttr`, JSONB MultiLang fields return raw JSON objects instead of localized strings, causing GraphQL serialization failures that abort entire queries via `GQLSchemaBase.execute`.

## Ejtmaa bridge bases

| Base | Path |
|---|---|
| `CustomerBridgeBase` | `backend/src/app/gql/bridges/customer/CustomerBridgeBase.ts` |
| `SupervisorBridgeBase` | `backend/src/app/gql/bridges/supervisor/SupervisorBridgeBase.ts` |

## Required methods

Every role bridge base must implement:

### `trans(cb, data?)`

Localization helper using `@nodejs/localization`.

### `toEnum(value, enumIdentify, data?)`

Converts a single enum value to `{value, label, icon}`.

### `toManyEnums(values, enumIdentify, data?)`

Converts enum value arrays to `{value, label, icon}[]`.

### `willPrepare(parent, prepareType)`

Base override (may be empty). Entity bridges override only for checks beyond `getRootOrmParent`.

### `loadAttr(options)` — critical

Handles before falling back to `super.loadAttr`:

1. MultiLang JSONB → localized string
2. MultiLang array → localized string array
3. Enum attrs → `{value, label, icon}` via `getAsSelect` / `getAsManySelect`
4. Default → `super.loadAttr`

## Required imports

```typescript
import {trans} from "@nodejs/localization";
import {LoadAttrOptions} from "@nodejs/gql";
import dottie from "dottie";
import {G_Tr} from "../../../../resources/trans/ar/general";
import {Nullable} from "../../../helpers/SureHelpers";
import pMap from "p-map";
import ModelBase from "../../../orm/models/Model";
```

## Interaction with `GQLSchemaBase.execute`

A single field serialization failure aborts the entire query response. The `loadAttr` override ensures values match SDL-declared types.

## Hard rules

1. Every role bridge base must have `loadAttr` override.
2. JSONB MultiLang fields declared as `String` in SDL must resolve to localized strings.
3. Enum attrs must use `getAsSelect` / `getAsManySelect`.
4. Include `willPrepare`, `trans`, `toEnum`, `toManyEnums` on every role base.

## Related

- `.cursor/rules/gql-schemas-bridges-general.mdc`
- `docs/platforms/backend/patterns/graphql-and-bridges.md`
