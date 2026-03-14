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
- [ ] **Session Statistics & Progress** — per-user breathing stats, streaks, progress tracking; built on top of the realtime foundation
- [ ] **Push Notifications** — session reminders, streak alerts

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
