---
name: aif-review
description: Perform code review on staged changes or a pull request. Auto-detects API (NestJS) or Mobile (Flutter) context from changed files and delegates to aif-review-api or aif-review-mobile with project-specific checklists.
argument-hint: "[PR number or empty]"
allowed-tools: Bash(git *) Bash(gh *) Read
---

# Code Review — Dispatcher

Detect project scope from changed files and delegate to the right specialist.

## Step 1: Get changed file paths

**Without arguments:**
- Run `git diff --cached --name-only`
- If empty → run `git diff --name-only`

**With PR number/URL:**
- Run `gh pr diff <number> --name-only`

## Step 2: Detect scope

| Changed files contain | Scope |
|-----------------------|-------|
| paths under `mind_api/` | API |
| paths under `mind_mobile/` | Mobile |
| both | API + Mobile |
| neither / unclear | Ask user |

## Step 3: Delegate

Read the appropriate sub-skill and follow its full instructions:

| Scope | Action |
|-------|--------|
| **API only** | Read `.claude/skills/aif-review-api/SKILL.md` and follow it |
| **Mobile only** | Read `.claude/skills/aif-review-mobile/SKILL.md` and follow it |
| **Both** | Follow `aif-review-api` for API files, then `aif-review-mobile` for mobile files |
| **Unclear** | Ask: "Which sub-project should I review — mind_api or mind_mobile?" |

Pass through any PR number argument to the sub-skill.
