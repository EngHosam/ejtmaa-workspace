# Agent Guidance

Use this root file only for concise project-wide agent guidance.

## Source Of Truth

- Cursor-specific persistent instructions live in `.cursor/rules/`.
- Task-specific agent workflows live in `.cursor/skills/<skill-name>/SKILL.md`.
- General project and architecture documentation lives in `docs/`.

## Working Rule

- Do not duplicate detailed skill contracts in this file.
- If guidance is specific to a task family, move it to the matching skill in `.cursor/skills/`.
- If guidance is persistent and repository-wide, move it to the correct rule in `.cursor/rules/`.
- Keep `AGENTS.md` short and high-signal.
