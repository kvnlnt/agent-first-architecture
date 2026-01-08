# Repository AI Rules

This file is authoritative.
All AI agents must read and follow it before performing any action.

If there is a conflict between instructions, follow this order:

1. policies/\*
2. repo.ai.md
3. packages/\*/ai.md
4. task file
5. agent instructions

If something is unclear, stop and ask. Do not guess.

---

## Repository Structure & Ownership

### Authored (human-owned, reviewed carefully)

- agents/
- config/
- packages/
- tests/
- tools/
- tasks/
- policies/
- public/ (authored static assets only)

### Generated (tool-owned, disposable)

- generated/

Rule:
If a file can be safely deleted and recreated, it belongs in `generated/`.
Generated files must never be manually edited.

---

## Agent Permissions

- Agents may only modify files explicitly within a task’s declared scope.
- Agents must not modify files outside the scope, even if changes seem helpful.
- Agents must not create new top-level directories.

---

## Packages

- All application code lives in `packages/`.
- Each package must contain an `ai.md` file describing its purpose and constraints.
- Agents must read `packages/*/ai.md` before modifying that package.
- Shared logic belongs in `packages/shared` (if present).

---

## Tests

- Tests live in `tests/`.
- Any behavior change requires corresponding test updates.
- Tests are authored code, not generated output.
- Do not update tests unless the task requires it.

---

## Generated Files

- Files in `generated/` may be overwritten freely.
- Agents should regenerate files rather than patching them.
- Diffs in `generated/` are informational only and not reviewed for logic.

---

## Tasks

- All agent work must originate from a task in `tasks/`.
- One task should result in one logical change.
- When complete, tasks should be moved to `tasks/done/`.

---

## Policies

- Policies in `policies/` override all other instructions.
- If a task conflicts with a policy, the agent must refuse the task.

---

## Safety Rules

Agents must not:

- Perform dependency upgrades unless explicitly instructed
- Make formatting-only changes
- Introduce breaking API changes without explicit approval
- Modify configuration files unless required by the task

---

## Default Behavior

- Prefer clarity over cleverness
- Make the smallest change that satisfies the task
- Do not refactor unrelated code
- Do not invent architecture or intent

When in doubt: stop.
