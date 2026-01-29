You are generating a fresh `tasks.md` for this repository to be used by a Ralph-Wiggum loop.

Inputs
- Read `implementation_plan.md` and use it as the source of truth for scope, ordering, and acceptance criteria.

Requirements
- Output only the full contents of `tasks.md` (no commentary or markdown fences).
- Use this exact structure:
  - `# Tasks: <project name> <short purpose>`
  - `### Global requirements (apply to all tasks)` with bullets
  - Numbered tasks: `### Task N: <Title>`
  - Each task must include:
    - `- Scope: ...` listing specific files/directories
    - `- Acceptance criteria:` followed by bullet points
- Keep tasks sequential and independent. The loop always works the first incomplete task.
- Make tasks bite-sized (1-2 focused changes).
- Acceptance criteria must be objective and verifiable. If tests/build are relevant, include exact commands.
- Preserve current behavior unless the goal explicitly changes it.
- Use ASCII only.

Process (do this before writing tasks)
- Read `implementation_plan.md` fully.
- Scan the repo to understand structure, build system, tests, and current behavior.
- Translate the plan into the smallest set of tasks needed to implement it.
- Order tasks so earlier tasks unblock later ones.
- If you need docs or config updates, include them as explicit tasks with scope and criteria.

Output checklist
- No extra sections beyond the format above.
- No TODO placeholders like “TBD” or “later”.
- No multi‑paragraph task descriptions; keep bullets concise.
