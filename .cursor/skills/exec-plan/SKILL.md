---
name: exec-plan
description: >-
  Executes an approved written plan with strict per-point compliance, converts it
  into independently editable review todos, summarizes each completed todo, and
  blocks progression until the user explicitly approves the next step. Use when
  the user names exec-plan with a plan document and wants gated step-by-step
  implementation without invention, exaggeration, or style drift from the repo.
---

# Exec plan

## When to Use

- The user invokes **`exec-plan`** together with an existing **execution plan** (for example output from `create-plan` or any written plan they paste).
- The user wants **strict adherence** to that plan, **independent todos** they can adjust per item, **line-by-line review with the agent**, a **summary after each todo**, and **no move to the next todo until they explicitly approve**.

## Instructions

1. **Read the plan carefully**  
   Parse the whole plan before changing code. Treat each requirement, ordering constraint, and verification note as binding unless the user amends it in the same thread.

2. **Mandatory compliance**  
   For every plan point: implement **only** what it demands. Forbidden: inventing features, over-engineering, expanding scope, or deviating from **repo-native** style, patterns, and contracts.

3. **Todos (independently editable)**  
   Translate the plan into a **sequenced todo list** where each item:
   - Has a **clear, standalone outcome** (so the user can edit one todo’s scope without rewriting the whole list),
   - States **dependencies** only when unavoidable,
   - Stays **small enough** to review in one pass.  
   Use the project’s todo mechanism when available; keep wording editable for the user.

4. **Step-by-step review**  
   Work **one todo at a time**. After finishing a todo, give a **short summary**: what was done, **which paths** changed, how to **verify**, and any **questions** if something in the plan was ambiguous.

5. **Approval gate (hard stop)**  
   Do **not** start the **next** todo (and do not mark the next item in progress) until the user gives **explicit approval** in the thread (for example: proceed / approved / next). If they want changes, revise the current or upcoming todos first, then continue.

6. **Conflicts**  
   If the plan conflicts with the codebase, governance, or another rule, **stop**, state the conflict, propose **concrete options**, and wait—do **not** pick silently.

7. **Verification**  
   Run only checks that **already exist** in the touched area (for example scripts in `package.json`). Do not invent new CI or build steps.

---

## Verbatim user contract (Arabic)

ادرس الخطة بدقة 
والتزم بكل نقطة فيها بشكل ملزم - لا اختراع ولا مبالغة ولا خروج عن المطلوب ولا عن الاسلوب المعتمد
ثم حولها الى todos قابله للتعديل بشكل مستقل لكل todo 
بحيث اقدر اراجعها معك خطوة بخطوة
وبعد انهاء كل todo اعطني ملخص ما تم فيها
ولا تنتقل الى ال todo التي بعدها الا بعد اعتمادي
