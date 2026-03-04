# Project: Mind

## Overview

Mind is a wellness/breathing app consisting of a NestJS REST API backend and a Flutter mobile app (iOS/Android). Users authenticate via Firebase, perform guided breathing sessions, and the app persists session history both locally (Drift) and remotely (PostgreSQL).

## Core Features

- Firebase-based authentication with JWT session management
- Guided breathing sessions with animated UI (shape morphing, physics-based motion)
- Breathing session history — CRUD, pagination, local cache + remote sync
- Offline-first: Drift (SQLite) local DB, synced to API on reconnect

## Tech Stack

### Backend (`mind_api/`)
- **Language:** TypeScript
- **Framework:** NestJS
- **Database:** PostgreSQL
- **ORM:** TypeORM (migrations-based, synchronize: false)
- **Auth:** Firebase Admin SDK (ID Token) + Passport JWT (access token) with JWT blacklist
- **Infra:** Docker (dev + prod), Makefile

### Mobile (`mind_mobile/`)
- **Language:** Dart
- **Framework:** Flutter 3+
- **Local DB:** Drift (SQLite ORM, code-gen based)
- **State:** Riverpod (presentation) + RxDart BehaviorSubject (domain)
- **Navigation:** GoRouter
- **HTTP:** Dio + AuthInterceptor
- **Flavors:** dev / prod (separate Firebase projects)

## Architecture Notes

### Backend
Modular Monolith. Each domain (auth, breath-sessions, firebase) is a self-contained NestJS feature module. Two-guard auth: `FirebaseAuthGuard` on `/auth/login`, `JwtAuthGuard` everywhere else. Controllers are thin — all logic in services.

### Mobile
Layered architecture: Repository → Notifier (domain) → Service → ViewModel → Screen + Coordinator. ViewModel is the module boundary; domain models never leak into presentation. DI is manual via `App.shared` singleton.

## Cross-project Coordination

- API changes first (migrations → controller → service), then mobile client
- DTO shapes must stay in sync: Dart models + Drift schema must match API responses
- Auth changes affect both `mind_api/src/auth/` and `mind_mobile/lib/Core/Api/AuthInterceptor.dart` + `lib/User/`

## Non-Functional Requirements

- Logging: NestJS built-in logger, configurable
- Error handling: Structured HTTP errors on API, typed domain events on mobile
- Security: JWT blacklist (PostgreSQL), Firebase token verification, HTTPS
