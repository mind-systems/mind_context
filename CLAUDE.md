# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository structure

This is a monorepo containing two independent projects, each with its own `CLAUDE.md`:

| Directory | Stack | Purpose |
|-----------|-------|---------|
| `mind_api/` | NestJS + TypeORM + PostgreSQL | Backend REST API |
| `mind_mobile/` | Flutter + Riverpod + Drift | iOS/Android mobile app |

Read each sub-project's `CLAUDE.md` before working within it.

## Cross-project coordination

When a task requires changes in **both** projects:
1. Create a task list covering both sides before implementing.
2. Implement API changes first (migrations → controller → service), then update the mobile client.
3. The mobile app communicates with the API via Dio (`lib/Core/Api/`). DTO shapes must stay in sync — changing an API response shape requires updating the corresponding Dart model and any Drift schema that caches it.
4. Auth token handling lives in `mind_api/src/auth/` and `mind_mobile/lib/Core/Api/AuthInterceptor.dart` + `lib/User/` — changes to the auth flow must be reflected in both.
