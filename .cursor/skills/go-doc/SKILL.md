---
name: go-doc
description: >-
  After a feature or plan is implemented, exhaustively inventories every new and
  modified tracked file, documents shipped behavior under docs/ with traceable
  paths and no silent omissions, and captures durable governance as new
  .cursor/rules or .cursor/skills when repeatable invariants emerged. Use when
  the user invokes go-doc post-execution for maintenance-grade documentation
  (future changes, debugging, operations) without guesswork or scope drift.
---

# Go doc

## When to Use

- The user says work is **done** and asks to **review new/modified files**, then **document** outcomes under `docs/` **by domain, behavior, and current structure**—with **full coverage** of the change set for later execution, edits, and incident response.
- The user also wants **new Cursor rules** in `.cursor/rules/` when important invariants or conventions were **learned during implementation**, and **new skills** in `.cursor/skills/` when a **repeatable workflow** was learned—without inventing content that was not evidenced by the change set.

## Instructions

1. **Establish the change set (exhaustive, mandatory)**  
   Use git (respect `.cursor/rules/workspace-git-repos.mdc`: `backend/`, `website/`, or workspace root as appropriate) to produce a **complete inventory** of **every** new, modified, renamed, or deleted **tracked** file that belongs to the completed work (use `git status`, `git diff --name-status`, and range diffs against the agreed base when needed).  
   - **Do not silently drop** a path: if something looks generated, binary, or noisy, still **account for it** in the inventory (mark it as generated/customer-only and state **why** it is not narrated line-by-line).  
   - Read each non-generated file that carries behavioral or contract meaning; pull minimal adjacent context only when required to describe truth accurately.

2. **Write `docs/` first (accurate, structured, maintenance-oriented)**  
   - Prefer **existing** doc trees and naming under `docs/`; extend them instead of inventing parallel hierarchies.  
   - Persisted authoritative documentation under `docs/` must stay **English** (see governance / constitution digest).  
   - Documentation exists for **future implementers**: execution, follow-up changes, debugging, and defect triage—so include **observable** contracts and runtime facts, not slogans.  
   - Document **what shipped**, with **no gaps** inside the change set: domain purpose; entry points; public surfaces (HTTP routes, GraphQL operations/types, events, jobs, CLI); persistence or config keys touched; authz/tenant boundaries; error/failure modes that the code actually handles; migrations/seeds/feature flags/env toggles when present in the diff.  
   - For seeds/demo fixtures: document **mechanism only** (entry console/action, init vs update, early-return/idempotency, which factories/mixins create rows, asset directories). Do **not** mirror demo payload values (emails, passwords, sample names, colors, status samples) into `docs/` — those stay in code.  
   - Seed naming: never document or introduce seed helpers named `plan`/`plans` (product-package collision). Note curated vs faker person-name policy when the change set established it.  
   - Also refresh **platform indexes** that list public surfaces when the change set expands them (e.g. website/cpanel overview GQL query lists, supervisor admin read-surface indexes), so indexes do not drift from the contract page.  
   - Provide a **traceability map**: either a single table or a short section that lists **each inventoried path** and where it is described (doc section anchor / subsection). If a path is intentionally excluded from deep narrative, record the **rationale** next to it.  
   - Link to **real paths** and symbols; do not document unimplemented or hypothetical behavior.

3. **Rules in `.cursor/rules/` (only when durable)**  
   Add or update an `.mdc` rule when the implementation revealed a **stable constraint** future agents must follow (security, contracts, file placement, naming, forbidden patterns).  
   - Match existing rule shape: frontmatter (`description`, `globs` when file-scoped, `alwaysApply` as needed), concise body, no duplicate of constitution text.  
   - **Do not** create a rule for one-off trivia, personal preference without repo evidence, or content already fully covered by an existing rule—**extend** the existing rule instead.

4. **Skills in `.cursor/skills/` (only when repeatable)**  
   Add a skill when the task produced a **reusable workflow** worth repeating (checklists, contracts, tool usage). Follow `.cursor/rules/skills-location-and-format.mdc`: folder name = YAML `name`, sections **When to Use** and **Instructions**, tight description.  
   - **Do not** create a skill that only restates generic advice or duplicates an existing skill; **update** the existing skill if the workflow belongs there.

5. **Discipline**  
   - **No invention**: docs, rules, and skills must reflect **observed** code and decisions from the implementation or explicit user statements in the thread.  
   - **No omission inside the change set**: completeness beats brevity for the shipped surface area. Prefer consolidation and tables over scatter, but **never** “skip the boring files” without an explicit, justified row in the inventory.  
   - **Lean governance outside docs**: add or extend rules/skills only when they prevent repeatable mistakes; still, **every** durable invariant learned during the work must land either in `docs/` (behavioral truth) or in `.cursor/rules/` / `.cursor/skills/` (repeatable enforcement), not only in chat memory.  
   - If domain ownership of a doc path is unclear, **ask** where the user wants the page to live before writing.

6. **Verification**  
   If documentation references commands or behavior, align with **existing** `package.json` scripts or repo-documented procedures only.

---

## Verbatim user contract (Arabic)

تم تنفيذ الخطة - راجع الملفات الجديدة والمعدلة
ثم وثق ما تم بشكل دقيق حسب الدومين والوظيفة والهيكل الحالي
الى docs
وكذلك اذا كان هناك قواعد مهمة تم فهمها خلال تنفيذ المهمة ، يجب انشاء rulls مناسبة لها في .cursor
وكذلك اذا كان هناك skills تم تعلمها خلال تنفيذ المهمة ، يجب انشاء skills مناسبة لها في .cursor

لازم يحصرها بدقة ولا يتجاهل اي شيء ابدا - التوثيق مهم للتنفيذ لاحقا او التعديل او اصلاح المشاكل ... الخ
