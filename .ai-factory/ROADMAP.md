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
- [ ] **Localized Transactional Emails** — OTP emails sent in the user's language; supported locales defined server-side
- [ ] **Locale Analytics** — track raw requested locales on registration with a counter to prioritize new localizations (этот пункт надо переделать на https://github.com/mind-systems/mind_api/issues/26)
- [ ] **Breath Session Complexity** — develop formulae for exercise complexity calculation and integrate
- [ ] **Session Statistics & Progress** — per-user breathing stats, streaks, progress tracking
- [ ] **Push Notifications** — session reminders, streak alerts
- [ ] **Public Session Sharing** — share breathing exercises via deep link

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
