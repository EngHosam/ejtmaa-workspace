---
name: backend-requester-whitespace-normalization
description: Audits and fixes requester input normalization for human/business text fields, preventing destructive whitespace cleanup and enforcing clean Joi patterns. Use when editing backend requesters, Joi validation, registration payload contracts, or MultiLang string write paths.
---

# Backend Requester Whitespace Normalization

## When to Use

- Use when modifying files under `backend/src/app/orchestrator/requesters/`.
- Use when changing Joi validation for `name`, `commercial_name`, or similar human/business text fields.
- Use when a single-input field is persisted as `MultiLangString`.
- Use when auditing `prepareCleanString(..., { cleanWhiteSpaces: true })` usage.

## Instructions

1. Read:
- `docs/platforms/backend/patterns/requesters-and-orchestration.md`
- `.cursor/rules/backend-requester-whitespace-normalization.mdc`

2. Apply normalization policy:
- Human/business text -> `joi.string().trim().min(...).required()` (keep internal spaces).
- Machine-safe identifiers (`email`, `mobile`) may use `cleanWhiteSpaces: true`.
- Avoid `.custom(...)` if Joi built-ins are enough; use `.external(...)` only for async checks.

3. MultiLang write-path policy:
- Keep client payload as single string.
- Map inline in requester to `{ ar: value, en: value }`.
- Do not introduce a generic helper for this mapping.

4. Consistency checks:
- If a value is normalized, uniqueness queries must use the normalized value.
- Ensure no persistence path collapses internal spaces in human text.

5. Verification:
- Run `yarn type-check` in `backend/`.
- Search for `cleanWhiteSpaces: true` and confirm contexts are policy-compliant.
