---
name: aif-plan-mind-api
description: >-
  Plan implementation for the mind_api backend (NestJS + TypeORM + PostgreSQL).
  Pre-scoped alias for aif-plan — skips sub-project detection, always targets mind_api/.ai-factory.
  Use when the task is clearly backend-only.
argument-hint: "[fast | full] [--parallel] <description>"
allowed-tools: Read Write Glob Grep Bash(git *) Bash(cd *) Bash(cp *) Bash(mkdir *) Bash(basename *) TaskCreate TaskUpdate TaskList AskUserQuestion Questions Task
disable-model-invocation: false
---

# Plan - mind_api (Backend)

Pre-scoped alias for `/aif-plan`. Sub-project is already known:

```
PLAN_ROOT = mind_api/.ai-factory
Sub-project = mind_api/ (NestJS + TypeORM + PostgreSQL)
```

**Skip Step 0.1 (Detect Sub-project Scope) entirely.** `PLAN_ROOT` is hardcoded above.

Then follow **all remaining steps from `/aif-plan`** starting at Step 0 (Load Project Context):

- Read `mind_api/.ai-factory/DESCRIPTION.md` and `.ai-factory/DESCRIPTION.md` for context
- Read `.ai-factory/ARCHITECTURE.md` if it exists
- Continue from Step 0.2 (Ensure Git Repository) through the end of the aif-plan workflow

See full workflow in `.claude/skills/aif-plan/SKILL.md`.
