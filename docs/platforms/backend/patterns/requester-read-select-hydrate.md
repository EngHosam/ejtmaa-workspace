# Requester Read — Select Hydrate Pattern

## 1) Why This Pattern Exists

Website multi-path edit forms hydrate via a requester `read` sub. Choice tiles (`FormChoiceField`) and entity pickers (`FormEntityPickerField`) already accept **SelectOption-shaped** store values (`{ value, label?, … }`).

Bare enum strings or bare FK ids on `read` force the form to re-map labels client-side and break picker hydrate. Requesters must return select-ready shapes for those fields.

## 2) Scope

Applies when a requester `read` returns values that the website form will bind to:

| Field kind | Website consumer | Read shape |
|---|---|---|
| Domain enum (`type`, status choice, …) | `FormChoiceField` | `toEnumForSelect(value, enumKey)` → `{ value, label, … }` |
| Related entity ref (channel, member, …) | `FormEntityPickerField` | `model.forSelect(lang)` → `{ value, label }` (or `null` when optional/absent) |

Does **not** replace:

- Write-path Joi: still `joi.select({ validValues })` (input may be string or SelectOption).
- Identity keys on `read`: still **no entity id echo** — see `.cursor/rules/requester-read-no-entity-id-echo.mdc`.
- Plain strings/numbers/booleans that are not select/picker fields.

## 3) Enum fields — `RequesterBase.toEnumForSelect`

**Helper:** `backend/src/app/orchestrator/RequesterBase.ts` → `toEnumForSelect(value, enumIdentify)`.

- `enumIdentify` is a key of `G_Tr["enums"]` (e.g. `"messageTemplateType"`, `"messageChannelType"`).
- Spreads the translated enum entry onto `{ value }`, so the client gets at least `{ value, label }`.
- Returns `undefined` when `value` is falsy — required enums on the row should always be present.

**ReadResult typing:** type the field as `SelectOption` (not the raw enum union) when `read` returns the select shape.

**Canonical:**

```ts
type: await this.toEnumForSelect(
    messageChannel.get("type"),
    "messageChannelType"
),
```

**Forbidden on read for choice-bound enums:**

```ts
type: messageChannel.get("type"), // bare enum string
```

Do **not** hand-build `{ value, label }` from `trans(...)` when `toEnumForSelect` covers the enum key.

## 4) Entity refs — per-model `forSelect(lang)`

Add an instance method on the **related** ORM model (not a generic Model base):

```ts
forSelect(_lang: string): SelectOption {
    return {
        value: this.get("id"),
        label: this.get("name") // or MultiLang resolve via lang when the display field is multilang
    };
}
```

**Rules:**

- Keep `forSelect` on the model that owns the display label (e.g. `MessageChannel.forSelect`).
- Signature keeps `lang` even when unused today — multilang labels need it later without caller churn.
- Requester `read` loads the association (or uses the already-resolved model) and calls `forSelect(this.context.lang())`.
- Optional FK: `channel?.forSelect(this.context.lang()) ?? null`.

**Canonical:**

```ts
const channel = await messageTemplate.getMessageChannel();
// …
messageChannel: channel?.forSelect(this.context.lang()) ?? null,
```

**Forbidden:**

```ts
messageChannel: messageTemplate.get("message_channel_id"), // bare id
messageChannel: {
    value: channelId,
    label: channel?.get("name") || `${channelId}` // hand-built in requester when forSelect exists
},
```

Do **not** invent a shared `Model.forSelect` base unless product explicitly asks for one; per-model methods match current ORM style.

## 5) Website readiness (consumer contract)

Website fields already tolerate select hydrate:

- `FormChoiceField` — selection compares / stores via option `value`; mapState may hold string or `{ value }`.
- `FormEntityPickerField` — store is `{ value, label, avatarUrl? }` (or `""` when cleared).

Authority for field chrome: `docs/platforms/website/flow-form-foundation.md` §3.5–3.6.

No website adapter is required for this pattern when those fields are used as-is.

## 6) Shipped references

| Surface | Path |
|---|---|
| Helper | `RequesterBase.toEnumForSelect` |
| Model | `MessageChannelModel.forSelect` |
| Enum read | `MessageChannelRequester.read` (`type`) |
| Enum + ref read | `MessageTemplateRequester.read` (`type`, `messageChannel`) |
| Domain contracts | `message-channel-domain.md`, `message-template-domain.md` |

## 7) Checklist (new or updated `read`)

1. List every `read` value bound to `FormChoiceField` or `FormEntityPickerField`.
2. Enums → `toEnumForSelect` + matching `general.enums` key; `ReadResult` uses `SelectOption`.
3. Entity refs → ensure `forSelect(lang)` on the related model; call it from `read`.
4. Keep identity out of `read` values (initProps only).
5. Write path stays `joi.select` / Opt helpers — do not change write contracts to require SelectOption-only input.
6. Verify: backend `yarn type-check`.

## 8) Governance pointers

- Rule: `.cursor/rules/requester-read-select-hydrate.mdc`
- Skill: `.cursor/skills/backend-requester-read-select-hydrate/SKILL.md`
- Related: `.cursor/rules/requester-read-no-entity-id-echo.mdc`, `.cursor/skills/backend-requester-governance/SKILL.md`
