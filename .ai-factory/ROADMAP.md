# Project Roadmap

> Mind — a wellness breathing app with guided sessions, animated UI, and cross-device user preferences.

## Milestones

- [x] **Project Infrastructure** — Docker, Makefile, CI/Jenkins, Swagger, migrations setup
- [x] **Passwordless Email Auth** — OTP flow (send-code → verify-code), JWT + blacklist (API + mobile)
- [x] **Google Sign-In** — server auth code flow, JWT exchange (API + mobile)
- [x] **Onboarding Flow** — first-run user experience
- [x] **Breathing Session Player** — guided phases (inhale / hold / exhale / rest) with 4-component animation system
- [x] **Breathing Session CRUD** — list, create, edit, delete with pagination (API + mobile)
- [x] **Offline-First Session Sync** — Drift local cache + remote API sync
- [x] **Breathing Session Constructor** — custom exercise builder
- [x] **User Profile API** — `PATCH /user` endpoint, `language` column on User entity, language included in all auth responses
- [x] **App Settings Infrastructure** — `AppSettingsRepository`, SharedPreferences for theme & language, first-launch locale detection, reactive `MyApp` (Riverpod `StateProvider`)
- [x] **Theme & Language UI** — Profile screen wired end-to-end: theme picker + language picker, reactive `MyApp` (ThemeMode + Locale)
- [x] **Language Server Sync** — language synced to server on login (server wins) and on preference change when authenticated
- [x] **App Localization** — translate all existing UI screens to supported languages (intl + ARB files); no new screens, just existing content
- [x] **Localized Transactional Emails** — OTP emails sent in the user's language; supported locales defined server-side
- [x] **Device Statistics** — `POST /device/ping` on every cold start; collect `installation_id`, platform, OS version, locale, timezone, screen info, app version; track unique installs and `last_seen_at` independent of auth (mind-systems/mind_api#26)
- [x] **Exercise Editing** — allow users to edit existing breathing exercises (name, phases, settings)
- [x] **Exercise Starring** — ability to star exercises so they appear in 'Starred' section of breath sessions list
- [x] **Session Sort Order** — fix seed script to insert sessions in reverse order (complex→simple) so simple sessions get newer `createdAt` timestamps and appear first; add `createdAt` to client domain model, API parsing, Drift schema, and use it for client-side sort within each list section (My / Starred / Shared) to match server order
- [x] **Public Session Sharing** — share breathing exercises via deep link
- [x] **Breath Session Complexity** — develop formulae for exercise complexity calculation and integrate
- [x] **JWT Session Management** — replace JWT blacklist with a `user_sessions` table (allow-list); store SHA-256 hash of the token (not raw JWT); no `expires_at` — session lives until explicitly deleted; login creates a row, logout deletes it; guard looks up hash on every request; client stores JWT encrypted on-device
- [x] **Realtime Socket Foundation** — persistent Socket.IO channel (one per user, lives for the app lifetime); JWT auth on handshake; ActivityEngine (start/pause/resume/end/abandon LiveSession); StateStore (in-memory socket + activity maps); server restart recovery; versioned message protocol; engine telemetry (breath phase instruction log); activity pause/resume
  - [x] API: Phases A–H complete (gateway, live session engine, stream engine, stats service)
  - [x] API: Activity pause/resume + telemetry lifecycle markers
  - [x] Mobile I-1: Transport layer (LiveSocketService, SocketConnectionCoordinator)
  - [x] Mobile I-2: Reconnect (exponential backoff, connectivity detection)
  - [x] Mobile I-3: Activity Integration (LiveSessionNotifier, ILiveSessionService, BreathSessionViewModel)
  - [x] Mobile I-4: Engine Telemetry (TelemetryBuffer, TelemetryService, phase streaming)
  - [x] Mobile I-5: Activity pause/resume wired end-to-end
- [x] **Session Statistics & Progress** — per-user breathing stats, streaks, progress tracking; built on top of the realtime foundation
- [ ] **Personal Access Tokens** `[API]` — create/revoke long-lived tokens for CLI/MCP access; `personal_access_tokens` table (hashed); accepted by JwtAuthGuard alongside regular JWTs; endpoints: `POST /auth/tokens`, `GET /auth/tokens`, `DELETE /auth/tokens/:id`
- [ ] **Exercise Time-of-Day Field** `[API]` — add nullable `timeOfDay` enum column (`morning | midday | evening`) to `breath_sessions`; migration, entity, DTOs, seed values
- [ ] **Suggestions Endpoint** `[API]` — `GET /breath_sessions/suggestions?timeOfDay=X`; returns random 3–4 of the user's sessions matching the requested time slot; client always sends morning/midday/evening (night maps to morning on the client side)
- [ ] **Mind MCP Server — Core** `[MCP]` — standalone TypeScript MCP package (`mind_mcp/`); stdio transport; env-based auth (`MIND_API_URL` + `MIND_JWT_TOKEN`); tool: `list_my_breath_sessions`
- [ ] **Mind MCP Server — AI Classification** `[MCP]` — tools: `classify_session_time_of_day` (Claude analyses description + phases, returns suggestion), `set_session_time_of_day` (PATCH API), `classify_all_sessions` (batch with confirmation)
- [ ] **Personal Access Tokens** `[Mobile]` — "MCP" cell in Profile screen; new MCP screen: token list, create button, one-time reveal modal, revoke; see `mind_mobile/.ai-factory/notes/mcp-screen.md`
- [ ] **Time-of-Day Suggestions Integration** `[Mobile]` — local time → morning/midday/evening helper; `fetchSuggestions` in IUserApi; Riverpod FutureProvider; widget wired to HomeScreen
- [ ] **Time-of-Day Suggestions Widget UI** `[Mobile]` — thin-bordered card, randomized localized title, horizontal auto-scrolling carousel (reverses at ends, stops on user interaction); see `mind_mobile/.ai-factory/notes/suggestions-widget.md`

## Completed

| Milestone | Date |
|-----------|------|
| Project Infrastructure | — |
| Passwordless Email Auth | — |
| Google Sign-In | — |
| Onboarding Flow | — |
| Breathing Session Player | — |
| Breathing Session CRUD | — |
| Offline-First Session Sync | — |
| Breathing Session Constructor | — |
| User Profile API | 2026-03-08 |
| App Settings Infrastructure | 2026-03-09 |
| Theme & Language UI | 2026-03-09 |
| Language Server Sync | 2026-03-12 |
| App Localization | 2026-03-12 |
| Localized Transactional Emails | 2026-03-12 |
| Device Statistics | 2026-03-12 |
| Exercise Editing | 2026-03-12 |
| Exercise Starring (API) | 2026-03-12 |
| Exercise Starring (mobile) | 2026-03-13 |
| Session Sort Order | 2026-03-13 |
| Public Session Sharing | 2026-03-13 |
| Breath Session Complexity | 2026-03-13 |
| JWT Session Management | 2026-03-13 |
| Realtime Socket Foundation | 2026-03-15 |
| Session Statistics & Progress | 2026-03-18 |
