# Project: Mind

## Overview

Mind is a wellness/breathing app consisting of a NestJS gRPC backend, a Flutter mobile app (iOS/Android), a static landing page, an MCP server for Claude Code integration, and a Flutter plugin wrapping the Neiry neurofeedback hardware SDK. Users authenticate via passwordless email (one-time code) or Google Sign-In, perform guided breathing sessions, and the app persists session history both locally (Drift) and remotely (PostgreSQL) with real-time sync over gRPC.

## Core Features

- Passwordless email authentication (one-time code) and Google Sign-In with JWT session management
- Guided breathing sessions with animated UI (shape morphing, physics-based motion)
- Breathing session history — CRUD, pagination, local cache + remote sync
- Offline-first: Drift (SQLite) local DB, synced to API on reconnect via gRPC streaming
- Real-time activity tracking: bidi-streaming gRPC session lifecycle + instruction stream
- Static landing page with Snake easter-egg (placeholder while real content is built)

## Tech Stack

### Backend (`mind_api/`)
- **Language:** TypeScript
- **Framework:** NestJS
- **Database:** PostgreSQL
- **ORM:** TypeORM (migrations-based, synchronize: false)
- **Transport:** gRPC (port 50051) via `@nestjs/microservices` + `@grpc/grpc-js`; single HTTP relay for Google OAuth browser callback
- **Auth:** `GrpcAuthInterceptor` validates JWT + `user_sessions` table on every gRPC call; passwordless OTP (`rpc SendCode` + `rpc VerifyCode`); Google Sign-In (`rpc GoogleAuth`)
- **Infra:** Docker (dev + prod), Makefile

### Mobile (`mind_mobile/`)
- **Language:** Dart
- **Framework:** Flutter 3+
- **Local DB:** Drift (SQLite ORM, code-gen based)
- **State:** Riverpod (presentation) + RxDart BehaviorSubject (domain)
- **Navigation:** GoRouter
- **Transport:** gRPC via `package:grpc` + `GrpcAuthInterceptor` (JWT in metadata)
- **Flavors:** dev / prod

### Landing (`mind_landing/`)
- **Stack:** Plain HTML/CSS/JS — single `index.html`, no build step, no dependencies
- **Current state:** Placeholder with a playable Snake game; real product content TBD

### MCP Server (`mind_mcp/`)
- **Language:** TypeScript
- **Transport:** stdio (MCP protocol)
- **Role:** Claude Code integration — exposes Mind API tools (breath sessions, auth tokens) to AI agents
- **Proto:** consumes `.proto` files from `mind_api/proto/`, regenerates TypeScript stubs via `npm run proto:gen`

### Neiry Kit (`neiry_kit/`)
- **Stack:** Flutter plugin (Dart + native iOS/Android bridges)
- **Role:** Wraps the Neiry/Capsule neurofeedback hardware SDK; tested via included example app before integrating into `mind_mobile`
- **Vendored binaries:** `official/iOS/CapsuleClient.framework` (XCFramework), `official/Android/CapsuleService.aar` + `devicedriver.aar`

## Architecture Notes

### Backend
Modular Monolith. Each domain (auth, breath-sessions, realtime, sync) is a self-contained NestJS feature module. All API traffic goes through gRPC; `GrpcAuthInterceptor` is the single auth boundary. Controllers are thin — all logic in services.

### Mobile
Layered architecture: Repository → Notifier (domain) → Service → ViewModel → Screen + Coordinator. ViewModel is the module boundary; domain models never leak into presentation. DI is manual via `App.shared` singleton. gRPC stubs generated from `mind_api/proto/`.

## Cross-project Coordination

- Proto contract changes start in `mind_api/proto/` — single source of truth
- After any proto change: implement in `mind_api` → copy `.proto` to consumers → consumers regenerate stubs
- Dart models + Drift schema must match API proto contracts
- Auth changes affect both `mind_api/src/users/` and `mind_mobile/lib/Core/Grpc/GrpcAuthInterceptor.dart`

## Non-Functional Requirements

- Logging: Winston with daily rotation (API); structured domain events (mobile)
- Error handling: gRPC status codes on API, typed domain events on mobile
- Security: JWT validated on every gRPC call via `user_sessions` table; SHA-256 hashing for OTP codes and PATs
