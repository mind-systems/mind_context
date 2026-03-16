# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository structure

This root repo is a coordination layer. `mind_api/` and `mind_mobile/` are **separate git repositories** (each has its own `.git` directory) that live inside it as subdirectories.

| Directory | Stack | Purpose |
|-----------|-------|---------|
| `mind_api/` | NestJS + TypeORM + PostgreSQL | Backend REST API — separate git repo |
| `mind_mobile/` | Flutter + Riverpod + Drift | iOS/Android mobile app — separate git repo |

**Git operations** (status, diff, commit, branch) must be run inside the respective subdirectory, not from the root. The root repo has no visibility into changes inside `mind_api/` or `mind_mobile/`.

Read each sub-project's `CLAUDE.md` before working within it.

## This repo is the orchestrator

The root of this repo is a **coordination layer** — it holds cross-project plans, roadmaps, AI context (`.ai-factory/`), and skills (`.claude/`). It does not contain runnable application code itself.

- Work scoped to the backend only → operate inside `mind_api/`, plans go to `mind_api/.ai-factory/`
- Work scoped to the mobile app only → operate inside `mind_mobile/`, plans go to `mind_mobile/.ai-factory/`
- Cross-project or architectural work → use the root `.ai-factory/`

AI skills (`aif-plan`, `aif-fix`, `aif-verify`) detect sub-project scope from task descriptions and route plan/fix files accordingly.

## Language

**All files must be written in English** — plans, roadmaps, DESCRIPTION.md, ARCHITECTURE.md, and any other generated or edited files. This applies regardless of the language used in conversation.

## Cross-project coordination

When a task requires changes in **both** projects:
1. Create a task list covering both sides before implementing.
2. Implement API changes first (migrations → controller → service), then update the mobile client.
3. The mobile app communicates with the API via Dio (`lib/Core/Api/`). DTO shapes must stay in sync — changing an API response shape requires updating the corresponding Dart model and any Drift schema that caches it.
4. Auth token handling lives in `mind_api/src/auth/` and `mind_mobile/lib/Core/Api/AuthInterceptor.dart` + `lib/User/` — changes to the auth flow must be reflected in both.
