# AGENTS.md

> Project map for AI agents. Keep this file up-to-date as the project evolves.

## Project Overview

Mind is a wellness/breathing app — NestJS REST API backend + Flutter mobile app. Users authenticate via Firebase, perform guided breathing sessions, and view session history synced between device and server.

## Tech Stack

| Layer | Stack |
|-------|-------|
| Backend | NestJS + TypeORM + PostgreSQL |
| Mobile | Flutter + Riverpod + Drift |
| Auth | Firebase (ID Token) → JWT (access) |
| Infra | Docker + Makefile |

## Project Structure

```
mind/
├── mind_api/           # NestJS backend (TypeScript)
│   ├── src/
│   │   ├── auth/       # Firebase guard, JWT guard, blacklist, user entity
│   │   ├── breath-sessions/  # Breathing session CRUD module
│   │   ├── firebase/   # FirebaseModule (global)
│   │   ├── migrations/ # TypeORM migration files
│   │   └── config/     # DB config, TypeORM CLI datasource
│   ├── Makefile        # Docker dev/prod helpers
│   └── CLAUDE.md       # Backend-specific agent instructions
│
├── mind_mobile/        # Flutter app (Dart)
│   ├── lib/
│   │   ├── Core/       # App singleton, Drift DB, Dio API client, router, env
│   │   ├── User/       # Auth state, UserNotifier, UserRepository
│   │   ├── BreathModule/  # Breathing session domain + all screens
│   │   │   ├── Core/   # BreathSessionNotifier, models, repositories
│   │   │   └── Presentation/  # BreathSessionsList + BreathSession screens
│   │   └── Views/      # Shared UI components
│   ├── docs/           # Architecture docs and Mermaid diagrams
│   └── CLAUDE.md       # Mobile-specific agent instructions
│
├── .ai-factory/        # AI agent context
│   ├── DESCRIPTION.md  # Project specification
│   └── ARCHITECTURE.md # Architecture decisions (if generated)
├── AGENTS.md           # This file
└── CLAUDE.md           # Root monorepo instructions
```

## Key Entry Points

| File | Purpose |
|------|---------|
| `mind_api/src/main.ts` | NestJS bootstrap |
| `mind_api/src/app.module.ts` | Root module, global config |
| `mind_mobile/lib/main_dev.dart` | Flutter dev entrypoint |
| `mind_mobile/lib/main_prod.dart` | Flutter prod entrypoint |
| `mind_mobile/lib/Core/App.dart` | Manual DI singleton |
| `mind_mobile/lib/router.dart` | GoRouter route definitions |

## Documentation

| Document | Path | Description |
|----------|------|-------------|
| Backend guide | `mind_api/CLAUDE.md` | Commands, architecture, patterns |
| Mobile guide | `mind_mobile/CLAUDE.md` | Commands, architecture, patterns |
| Notifier pattern | `mind_mobile/docs/core/notifier-pattern.md` | Domain state pattern |
| JWT lifecycle | `mind_mobile/docs/core/jwt-authentication.md` | Auth token flow |
| Global listeners | `mind_mobile/docs/core/global-listeners.md` | Event listener system |
| Session lifecycle | `mind_mobile/docs/breath/session/session-lifecycle.md` | Session completion flow |

## AI Context Files

| File | Purpose |
|------|---------|
| `AGENTS.md` | This file — project structure map |
| `CLAUDE.md` | Monorepo instructions and cross-project coordination rules |
| `.ai-factory/DESCRIPTION.md` | Project specification and tech stack |
| `mind_api/CLAUDE.md` | Backend-specific instructions |
| `mind_mobile/CLAUDE.md` | Mobile-specific instructions |

## Roadmap Context

This root context is used for **strategic planning**:
- Create roadmap items here with `/aif-roadmap`
- For each roadmap item, create implementation plans in the target sub-project:
  - Backend tasks → open Claude Code in `mind_api/` and run `/aif-plan`
  - Mobile tasks → open Claude Code in `mind_mobile/` and run `/aif-plan`
  - Cross-cutting tasks → coordinate both, API first then mobile

## Cross-project Change Rules

1. API changes first: migrations → controller → service
2. Then update mobile: Dart models → Drift schema → ViewModel DTOs
3. Auth changes affect both `mind_api/src/auth/` and `mind_mobile/lib/Core/Api/AuthInterceptor.dart`
