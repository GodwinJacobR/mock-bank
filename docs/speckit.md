# Spec Kit

GitHub Spec Kit (`specify-cli` 1.3.2) is set up for Claude Code. Agent: `claude`, scripts: PowerShell.

## Install (once per machine)

```
uv tool install specify-cli
```

Setup was done with `specify init --here --force --non-interactive --integration claude --script ps`.

## Where files live

- `.claude/skills/speckit-*/` : the slash commands (Claude Code skills).
- `.specify/memory/constitution.md` : project constitution (filled in with our rules).
- `.specify/templates/`, `.specify/scripts/powershell/`, `.specify/workflows/` : Spec Kit internals. Do not hand-edit.
- `specs/<NNN-feature>/` : per-feature `spec.md`, `plan.md`, `tasks.md` created by Spec Kit.

## Commands, in order

Run inside Claude Code, from the repo root or a worktree:

1. `/speckit-constitution` : review or amend the constitution.
2. `/speckit-specify` : write the feature spec.
3. `/speckit-clarify` (optional) : resolve ambiguities.
4. `/speckit-plan` : implementation plan.
5. `/speckit-checklist` (optional) : requirement quality checklist.
6. `/speckit-tasks` : task list.
7. `/speckit-analyze` (optional) : consistency check.
8. `/speckit-implement` : build it.
9. `/speckit-converge` : append remaining work as tasks.

## Coexisting with existing files

- Our own specs (`specs/000-architecture.md`, `specs/001a`..`001g`) stay as they are. They are flat files; Spec Kit uses folders (`specs/NNN-name/`). Both live in `specs/`.
- Pick feature numbers that do not collide with existing ones. Treat `000-architecture.md` as the source of architecture rules; reference it from plans.
- `TASKS.md` remains the top-level tracker. Spec Kit's `tasks.md` is per feature; link or summarize it in `TASKS.md`.
- Spec Kit may try to create git branches itself. Our rule is a worktree and a feature branch per task, then a PR.
- `.claude/` may hold local settings; do not commit credentials.
