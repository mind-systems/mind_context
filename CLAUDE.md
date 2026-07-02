# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Deployment

See [docs/deployment.md](docs/deployment.md) — server connection, deploy script, env files, ports.

## Repository structure

This root repo is a coordination layer. All sub-directories below are **separate git repositories** (each has its own `.git` directory) that live inside it as subdirectories.

| Directory | Stack | Purpose |
|-----------|-------|---------|
| `mind_api/` | NestJS + TypeORM + PostgreSQL | Backend REST API — separate git repo |
| `mind_mobile/` | Flutter + Riverpod + Drift | iOS/Android mobile app — separate git repo |
| `mind_web/` | React + Vite + TypeScript + ECharts | Browser dashboard for historical session data — separate git repo |
| `mind_landing/` | Plain HTML/CSS/JS | Static landing page — separate git repo |
| `mind_mcp/` | TypeScript + MCP stdio | MCP server for Claude Code — separate git repo |
| `neiry_kit/` | Flutter plugin | Neiry neurofeedback SDK wrapper — separate git repo |
| `camera_ppg_kit/` | Flutter plugin | Camera+torch contact-PPG heart-rate source — separate git repo; **not wired into root orchestration**, its planning stays local (see its `CLAUDE.md`) |

**Git operations** (status, diff, commit, branch) must be run inside the respective subdirectory, not from the root. The root repo has no visibility into changes inside the sub-repos.

Read each sub-project's `CLAUDE.md` before working within it.

## This repo is the orchestrator

The root of this repo is a **coordination layer** — it holds cross-project plans, roadmaps, AI context (`.ai-factory/`), and skills (`.claude/`). It does not contain runnable application code itself.

## Scope routing

Work scoped to one sub-repo → operate inside it; its plans and roadmaps go to that repo's own `.ai-factory/`. Cross-project or architectural work → the root `.ai-factory/`. `camera_ppg_kit` is outside root orchestration — its planning stays local to that repo.

### Routing tables (from the root)

| Argument prefix | Target |
|---|---|
| `api` | `mind_api/.ai-factory/` |
| `mobile` | `mind_mobile/.ai-factory/` |
| `web` | `mind_web/.ai-factory/` |
| `mcp` | `mind_mcp/.ai-factory/` |
| `landing` | `mind_landing/.ai-factory/` |
| `neiry` | `neiry_kit/.ai-factory/` |

| Keywords in description | Target |
|---|---|
| NestJS, TypeORM, PostgreSQL, migration, endpoint, controller, service, realtime, sync | `mind_api/.ai-factory/` |
| Flutter, Dart, Drift, Riverpod, widget, screen, GoRouter, BCI screen, biometrics | `mind_mobile/.ai-factory/` |
| React, Vite, ECharts, TailwindCSS, web dashboard, browser, LoginPage, SessionsPage | `mind_web/.ai-factory/` |
| MCP, tool, stdio, classify, PAT, personal access token (server context) | `mind_mcp/.ai-factory/` |
| landing, landing page, HTML, CSS, Snake, placeholder | `mind_landing/.ai-factory/` |
| Neiry, Capsule, platform channel, EEG SDK, plugin native code | `neiry_kit/.ai-factory/` |
| architecture, cross-project | root `.ai-factory/` |

If detection is ambiguous, ask:
```
Which sub-project is this for?
1. mind_api (NestJS backend)
2. mind_mobile (Flutter app)
3. mind_web (React web dashboard)
4. mind_mcp (MCP server)
5. mind_landing (static landing page)
6. neiry_kit (Flutter BCI plugin)
7. Root / cross-project
```

## Language

**All files must be written in English** — plans, roadmaps, DESCRIPTION.md, ARCHITECTURE.md, and any other generated or edited files. This applies regardless of the language used in conversation.

## Proto contract ownership

**`mind_api/proto/` is the single source of truth for all `.proto` files.**

- No other project may create or modify `.proto` files.
- Any contract change starts in `mind_api/proto/`, then each consumer (`mind_mcp`, `mind_mobile`) copies the updated files and regenerates stubs.
- Do not use symlinks — they break when repos are cloned independently.
- Change order: `mind_api/proto/` → implement in `mind_api` → copy proto to consumers → consumers regenerate and implement.

## Cross-project coordination

When a task requires changes in **both** projects:
1. Create a task list covering both sides before implementing.
2. Implement API changes first (migrations → controller → service), then update the mobile client.
3. The mobile app communicates with the API via gRPC (`lib/Core/Grpc/` — `GrpcClient`, `GrpcAuthInterceptor`); contract changes flow through proto regeneration (see Proto contract ownership). A thin HTTP layer (`lib/Core/Api/`) remains for PAT/Sync helpers — its DTO shapes and any Drift schema that caches them must be updated by hand when the API response shape changes.
4. Auth token handling lives in `mind_api/src/users/` (auth module) and `mind_mobile/lib/Core/Grpc/GrpcAuthInterceptor.dart` + `lib/User/` — changes to the auth flow must be reflected in both.
