---
name: aif-plan
description: Plan implementation for a feature or task. Always creates a named plan file with full exploration. Use when user says "plan", "new feature", "start feature", "create tasks".
argument-hint: "<description>"
allowed-tools: Read Write Glob Grep Bash(git *) Bash(mkdir *) TaskCreate TaskUpdate TaskList AskUserQuestion Task
disable-model-invocation: false
---

# Plan - Implementation Planning

Creates a named plan file with full codebase exploration and preference questions.

## Workflow

### Step 0: Load Project Context

Read `.ai-factory/DESCRIPTION.md` and `.ai-factory/ARCHITECTURE.md` if they exist. Use this context when exploring the codebase and writing task descriptions.

### Step 0.1: Detect Sub-project Scope

This repository is a **monorepo**. Plans must live next to the code they describe.

**Detection rules (in order):**

1. **Explicit sub-project name in description** — `mind_api` or `mind_mobile` → use that sub-project.
2. **Technology keywords:**
   - NestJS, TypeORM, PostgreSQL, Swagger, Passport, Makefile, Docker → `mind_api/`
   - Flutter, Dart, Drift, Riverpod, widget, screen, GoRouter, Dio → `mind_mobile/`
3. **Cross-project or roadmap** → use root `.ai-factory/`

**Set `PLAN_ROOT`:**

```
mind_api task    → PLAN_ROOT = mind_api/.ai-factory
mind_mobile task → PLAN_ROOT = mind_mobile/.ai-factory
cross-project    → PLAN_ROOT = .ai-factory
```

### Step 0.2: Ensure Git Repository

```bash
git rev-parse --is-inside-work-tree 2>/dev/null || git init
```

---

### Step 1: Quick Reconnaissance

Launch 1-2 Explore agents to understand the relevant codebase area:

```
Task(subagent_type: Explore, model: sonnet, prompt:
  "In [project root], find files and modules related to [feature domain].
   Report: key directories, relevant files, existing patterns, integration points.
   Thoroughness: quick. Return a structured summary, not file contents.")
```

Skip if `DESCRIPTION.md` already provides sufficient context.

### Step 1.2: Ask Preferences

```
AskUserQuestion: Before we start:

1. Should I write tests for this feature?
   - [ ] Yes
   - [ ] No

2. Update documentation after implementation?
   - [ ] Yes (/aif-docs)
   - [ ] No

3. Any specific requirements or constraints?
```

---

### Step 2: Analyze Requirements

From the description identify:
- Core functionality to implement
- Components/files that need changes
- Dependencies between tasks
- Edge cases to handle

If requirements are ambiguous, ask clarifying questions before continuing.

### Step 3: Explore Codebase

Launch 2-3 Explore agents in parallel:

```
Agent 1 — Architecture & affected modules:
Task(subagent_type: Explore, model: sonnet, prompt:
  "Find files and modules related to [feature domain]. Map directory structure,
   key entry points, and module interactions. Thoroughness: medium.")

Agent 2 — Existing patterns & conventions:
Task(subagent_type: Explore, model: sonnet, prompt:
  "Find examples of similar functionality in the project.
   Show patterns for [relevant patterns]. Thoroughness: medium.")

Agent 3 — Dependencies & integration points (if needed):
Task(subagent_type: Explore, model: sonnet, prompt:
  "Find all files that import/use [module/service]. Identify integration points
   and side effects. Thoroughness: medium.")
```

Use recon from Step 1 as starting point, focus agents on gaps.

Fallback: use Glob/Grep/Read directly if Task tool unavailable.

**Synthesize:** which files to create/modify, patterns to follow, risks.

### Step 4: Create Task Plan

Create tasks using `TaskCreate`.

**Guidelines:**
- Each task completable in one focused session
- Ordered by dependency
- Include file paths
- Specific deliverables, not vague descriptions

Use `TaskUpdate` to set `blockedBy` relationships.

### Step 5: Save Plan to File

Save to `<PLAN_ROOT>/plans/<NN>-<slug>.md` where:
- `<slug>` is derived from the description (lowercase, hyphens)
- `<NN>` is a zero-padded two-digit sequence number (`01`, `02`, `03` …)

To determine `<NN>`, find the highest existing `NN` prefix among files matching `[0-9][0-9]-*.md` in `<PLAN_ROOT>/plans/` and add 1. If no numbered files exist yet, start at `01`.

```bash
mkdir -p <PLAN_ROOT>/plans
```

**Plan file must include:**
- Title and creation date
- `Settings` section (Testing, Docs)
- `Tasks` section grouped by phases

Use the template in `references/TASK-FORMAT.md`.

### Step 6: Next Steps

```
Plan created with [N] tasks.
Plan file: <PLAN_ROOT>/plans/<NN>-<slug>.md

To start implementation:
/aif-implement

To view tasks:
/tasks
```

Suggest freeing context after planning:

```
AskUserQuestion: Free up context?
1. /clear — Full reset (recommended)
2. /compact — Compress history
3. Continue as is
```

---

## Task Requirements

Every `TaskCreate` item MUST include:
- Clear deliverable and expected behavior
- File paths to change/create
- Dependency notes when applicable

See `references/TASK-FORMAT.md` for examples.

## Rules

1. **NO tests if user said no**
2. **NO reports** — no summary tasks at the end
3. **Actionable tasks** — clear deliverable per task
4. **Right granularity** — not too big, not too small
5. **Dependencies matter** — sequential ordering
6. **Include file paths**
7. **Plan file location** — always `<PLAN_ROOT>/plans/<NN>-<slug>.md` with zero-padded sequence number

## Plan File Handling

| Scope | PLAN_ROOT |
|-------|-----------|
| Backend (NestJS, TypeORM…) | `mind_api/.ai-factory` |
| Mobile (Flutter, Drift…) | `mind_mobile/.ai-factory` |
| Cross-project or roadmap | `.ai-factory` |

Plan files live at `<PLAN_ROOT>/plans/<slug>.md` and survive across sessions.
