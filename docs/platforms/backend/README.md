# Backend Documentation Index

## Overview

- [`overview.md`](overview.md) — Platform architecture and active domain surface.

## Contracts

| Document | Scope |
|---|---|
| [`http-and-requesters.md`](contracts/http-and-requesters.md) | HTTP mounts, middleware, requester dispatch |
| [`graphql-and-types.md`](contracts/graphql-and-types.md) | Customer + supervisor GQL schemas and codegen |
| [`client-portal-http-website.md`](contracts/client-portal-http-website.md) | `/website` client portal contract |
| [`account-settings-requesters.md`](contracts/account-settings-requesters.md) | Customer account settings on `/website` |
| [`external-http-mount-and-myfatoorah-callbacks.md`](contracts/external-http-mount-and-myfatoorah-callbacks.md) | External HTTP and payment callbacks |
| [`supervisor-admin-read-surfaces.md`](contracts/supervisor-admin-read-surfaces.md) | Supervisor admin read surfaces |
| [`supervisor-customers-and-stats.md`](contracts/supervisor-customers-and-stats.md) | Customer management and stats |
| [`organization-domain.md`](contracts/organization-domain.md) | Organization tenant ORM + GQL + seed |
| [`member-domain.md`](contracts/member-domain.md) | Member ORM + customer GQL reads |
| [`message-template-domain.md`](contracts/message-template-domain.md) | MessageTemplate ORM + customer GQL reads |
| [`meeting-domain.md`](contracts/meeting-domain.md) | Meeting ORM + customer GQL reads |
| [`socket-event-mirroring.md`](contracts/socket-event-mirroring.md) | Socket notify events |

## Patterns

| Document | Scope |
|---|---|
| [`graphql-and-bridges.md`](patterns/graphql-and-bridges.md) | GQL schema and bridge patterns |
| [`requesters-and-orchestration.md`](patterns/requesters-and-orchestration.md) | Requester orchestration |
| [`gql-source-corpus.md`](patterns/gql-source-corpus.md) | Ejtmaa GQL source corpus |
| [`gql-pattern-atlas.md`](patterns/gql-pattern-atlas.md) | Reusable GQL pattern families |
| [`gql-role-bridge-base-contract.md`](patterns/gql-role-bridge-base-contract.md) | Role bridge base contract |
| [`gql-schema-bridge-authoring-standard.md`](patterns/gql-schema-bridge-authoring-standard.md) | Schema/bridge authoring |
| [`gql-worked-reference-pack.md`](patterns/gql-worked-reference-pack.md) | Worked reference pack |
| [`gql-union-routing-reference.md`](patterns/gql-union-routing-reference.md) | Union routing |
| [`gql-constitution-map.md`](patterns/gql-constitution-map.md) | GQL constitution map |
| [`gql-complex-problems-cookbook.md`](patterns/gql-complex-problems-cookbook.md) | Complex problems cookbook |
| [`gql-complex-problems-playbooks.md`](patterns/gql-complex-problems-playbooks.md) | Complex problems playbooks |
| [`gql-anti-patterns-blacklist.md`](patterns/gql-anti-patterns-blacklist.md) | Anti-patterns blacklist |
| [`gql-refusal-matrix.md`](patterns/gql-refusal-matrix.md) | Refusal matrix |
| [`gql-evaluation-suite.md`](patterns/gql-evaluation-suite.md) | Evaluation suite |
| [`gql-scoring-rubric.md`](patterns/gql-scoring-rubric.md) | Scoring rubric |
| [`orm-constitution-map.md`](patterns/orm-constitution-map.md) | ORM constitution map |
| [`orm-model-authoring-standard.md`](patterns/orm-model-authoring-standard.md) | ORM authoring standard |
| [`orm-model-baseline-audit.md`](patterns/orm-model-baseline-audit.md) | ORM baseline audit |
| [`orm-model-compliance-checklists.md`](patterns/orm-model-compliance-checklists.md) | ORM compliance checklists |
| [`orm-model-worked-examples.md`](patterns/orm-model-worked-examples.md) | ORM worked examples |
| [`models-owners-abilities-security.md`](patterns/models-owners-abilities-security.md) | Models, owners, abilities |
| [`scheduler-console-seed-db.md`](patterns/scheduler-console-seed-db.md) | Scheduler, console, seed |

## Modules

- [`modules/runtime-integrations.md`](modules/runtime-integrations.md) — Provider integrations.

## Playbooks

- [`playbooks/add-update-fix.md`](playbooks/add-update-fix.md) — Operational checklists.

## Invariants

- [`../../invariants/backend.md`](../../invariants/backend.md)
