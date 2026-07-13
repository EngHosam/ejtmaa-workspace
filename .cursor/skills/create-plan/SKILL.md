---
name: create-plan
description: >-
  Produces a tight execution plan and phased todos for precision tasks across
  website, cpanel, and backend, with minimal diffs, repo-native patterns, edge-case
  handling, conflict surfacing, and verification against existing scripts and
  governance. Use when the user invokes this skill with a concrete task, asks for
  a mandatory cross-surface execution plan, or requires reviewable per-todo
  deliverables with explicit escalation instead of assumptions.
---

# Ejtmaa cross-surface task plan

## When to Use

- The user names this skill and pastes a task that may touch **website**, **cpanel**, and/or **backend** (or any subset), and wants a **detailed execution plan** another agent can follow.
- The user wants work **split by location** with **logically independent todos** so they can review after each todo.
- The user demands **minimal change**, **no invented requirements**, and **explicit questions** when the repo does not show how to resolve impact.

## Instructions

1. **Scope reads (task-only)**  
   Inspect only what the task needs from:
   - `.cursor/**` (rules, skills)
   - `docs/**`
   - relevant **source** paths  
   Do not load unrelated files or broad scans "for context."

2. **Repo as sole style authority**  
   Treat this repository as the canonical reference for patterns, naming, layering, and UI/API contracts. Match existing code; do not introduce parallel conventions.

3. **Edge cases and conflicts**  
   Enumerate expected use cases and failure modes; ensure the plan's code paths handle them without contradicting existing behavior or invariants. If two approaches conflict, pick neither silently—surface it (see step 6).

4. **Prior work outside this repo**  
   If nothing similar exists locally, use cross-project retrieval per `.cursor/rules/use-code-search-for-projects.mdc` (e.g. `code-search` MCP: `search_guides`, `search_code`, `open_file`). Use results only as **style/pattern hints**; adapt to **this** repo's scope and contracts.

5. **Todos: reviewable and independent**  
   Structure todos so each item is:
   - **Logically standalone** (clear feature slice or vertical slice),
   - **Deliverable** (can be reviewed and approved before the next),
   - **Scoped** to one platform when possible.

6. **Escalation**  
   When certainty is below high confidence, stop and ask. Do not invent architecture, contracts, or domain features.

7. **Verification**  
   Run only `package.json` scripts that already exist for affected platforms. Do not invent verification tooling.

## Platform scope

- `backend/` — requesters, ORM, GQL, HTTP mounts `/website` and `/cpanel`
- `cpanel/` — Supervisor SSR frontend
- `website/` — Customer SSR frontend

## Output format

Write a `.plan.md` with frontmatter todos (`id`, `content`, `status: pending`) and sections:
- Decisions locked
- Per-todo deliverables with file paths
- Verification per platform
- Escalation points
