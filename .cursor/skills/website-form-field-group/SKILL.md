---
name: website-form-field-group
description: >-
  Wraps related website form fields in FormFieldGroup (titled card + optional
  subTitle). Use when adding or moving a cluster of fields that share one
  explanation — not a single-field wrapper and not a page heading.
---

# Website FormFieldGroup

## When to Use

- Two or more form fields belong together and share one title or explanation.
- Replacing a one-off bordered `Col` around fields.
- Moving group copy off a single field `subTitle`.

## Instructions

1. Import `FormFieldGroup` from `website/src/app/ui/components/form/FormFieldGroup.tsx`.
2. Pass `title` and optional `subTitle` from the page translator. Put the fields in `children`.
3. Do not duplicate card chrome (`cardBackground` / `shellBorder` / `card.radius` / `p={1.15}`) at the call site.
4. Do not hang the group sentence on the first field’s `subTitle`.
5. Do not use `SectionHeading` (page chrome) or `FormInputWrapper` (one field) for the cluster.
6. Organization identity colors is the canonical consumer — `flow-customer-organization.md` §4.1. Meeting paint remap stays in `useOrganization` (W62); group copy explains it, pickers stay stored hex.
7. Update `flow-form-foundation.md` §3.4b and the consumer flow in the same task.

## Canonical reference

- Component: `website/src/app/ui/components/form/FormFieldGroup.tsx`
- Rule: `.cursor/rules/website-form-field-group.mdc`
- Foundation: `docs/platforms/website/flow-form-foundation.md` §3.4b
