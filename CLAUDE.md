# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository structure

This root repo is a coordination layer. `mind_api/`, `mind_mobile/`, and `mind_landing/` are **separate git repositories** (each has its own `.git` directory) that live inside it as subdirectories.

| Directory | Stack | Purpose |
|-----------|-------|---------|
| `mind_api/` | NestJS + TypeORM + PostgreSQL | Backend REST API — separate git repo |
| `mind_mobile/` | Flutter + Riverpod + Drift | iOS/Android mobile app — separate git repo |
| `mind_landing/` | Plain HTML/CSS/JS | Static landing page — separate git repo |

**Git operations** (status, diff, commit, branch) must be run inside the respective subdirectory, not from the root. The root repo has no visibility into changes inside `mind_api/` or `mind_mobile/`.

Read each sub-project's `CLAUDE.md` before working within it.

## This repo is the orchestrator

The root of this repo is a **coordination layer** — it holds cross-project plans, roadmaps, AI context (`.ai-factory/`), and skills (`.claude/`). It does not contain runnable application code itself.

## Scope routing

- Work scoped to the backend only → operate inside `mind_api/`, plans go to `mind_api/.ai-factory/`
- Work scoped to the mobile app only → operate inside `mind_mobile/`, plans go to `mind_mobile/.ai-factory/`
- Work scoped to MCP server → operate inside `mind_mcp/`, plans go to `mind_mcp/.ai-factory/`
- Work scoped to the landing page → operate inside `mind_landing/`, plans go to `mind_landing/.ai-factory/`
- Cross-project or architectural work → use the root `.ai-factory/`

### `/aif-plan` routing rules

When `/aif-plan` is run, first check the current working directory:

- **CWD is inside a sub-project** (`mind_api/`, `mind_mobile/`, `mind_mcp/`, `mind_landing/`) → save plan to `.ai-factory/plans/` relative to CWD. No detection needed.
- **CWD is the monorepo root** → detect target sub-project from the task description:

| Keywords in description | Target | Plan path |
|---|---|---|
| NestJS, TypeORM, PostgreSQL, Swagger, Passport, migration, endpoint, controller, service | `mind_api` | `mind_api/.ai-factory/plans/` |
| Flutter, Dart, Drift, Riverpod, widget, screen, GoRouter, Dio | `mind_mobile` | `mind_mobile/.ai-factory/plans/` |
| MCP, tool, stdio, classify, PAT, personal access token (server context) | `mind_mcp` | `mind_mcp/.ai-factory/plans/` |
| landing, landing page, HTML, CSS, Snake, placeholder | `mind_landing` | `mind_landing/.ai-factory/plans/` |
| architecture, roadmap, cross-project, or ambiguous | root | `.ai-factory/plans/` |

If detection is ambiguous, ask:
```
Which sub-project is this plan for?
1. mind_api (NestJS backend)
2. mind_mobile (Flutter app)
3. mind_mcp (MCP server)
4. mind_landing (static landing page)
5. Root / cross-project
```

### `/aif-roadmap` routing rules

Default: works relative to CWD — no detection needed when already inside a sub-project.

When run from the monorepo root, or when the user explicitly names a sub-project in the argument (e.g. `/aif-roadmap api` or `/aif-roadmap api check`):

| Argument prefix | Target |
|---|---|
| `api` | `mind_api/.ai-factory/ROADMAP.md` |
| `mobile` | `mind_mobile/.ai-factory/ROADMAP.md` |
| `mcp` | `mind_mcp/.ai-factory/ROADMAP.md` |
| `landing` | `mind_landing/.ai-factory/ROADMAP.md` |
| no prefix / `check` / vision text | `.ai-factory/ROADMAP.md` relative to CWD |

Strip the sub-project prefix before processing the remaining argument (e.g. `api check` → mode `check` for api).

## Language

**All files must be written in English** — plans, roadmaps, DESCRIPTION.md, ARCHITECTURE.md, and any other generated or edited files. This applies regardless of the language used in conversation.

## Cross-project coordination

When a task requires changes in **both** projects:
1. Create a task list covering both sides before implementing.
2. Implement API changes first (migrations → controller → service), then update the mobile client.
3. The mobile app communicates with the API via Dio (`lib/Core/Api/`). DTO shapes must stay in sync — changing an API response shape requires updating the corresponding Dart model and any Drift schema that caches it.
4. Auth token handling lives in `mind_api/src/auth/` and `mind_mobile/lib/Core/Api/AuthInterceptor.dart` + `lib/User/` — changes to the auth flow must be reflected in both.
