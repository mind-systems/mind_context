# Project Roadmap

> Replace Firebase auth with passwordless email-code auth, then ship a fully self-contained wellness/breathing app with offline-first sync.

## Context

We are replacing Firebase authentication with our own passwordless auth via email. User enters email → receives a letter with a link → link opens the app → app verifies the code and receives JWT tokens. Firebase is fully removed from both projects.

## Milestones

### Step 1 — Database: `auth_codes` table

- [ ] **auth_codes table**

Create the `auth_codes` table. Fields: `id`, `email`, `code_hash` (sha256 of the 9-digit code), `created_at`, `expires_at` (now + 15 minutes), `used` boolean.
Index on `code_hash` — used for lookup during validation. Index on `expires_at` — used for cleanup of expired records.

---

### Step 2 — Endpoint `POST /auth/send-code`

- [ ] **POST /auth/send-code**

Accepts email. Generates a code via `crypto.randomInt(100000000, 999999999)` — produces exactly 9 digits, cryptographically secure. Hashes it: `crypto.createHash('sha256').update(code).digest('hex')`. Writes record to the table. If a live record for this email already exists — either delete the old one, or check that at least 60 seconds have passed since the last send, otherwise return 429. This protects against self-spam.

---

### Step 3 — Email delivery via Resend

- [ ] **Resend email integration**

Register on Resend, add domain `mind-awake.life`. Cloudflare DNS gets three records that Resend will show: SPF, DKIM, and DMARC. Add them, wait for verification (usually a few minutes). Call their SDK from the server; the email body contains the link `https://mind-awake.life/auth?code=XXXXXXXXX`. This is an HTTPS deeplink that is already configured to open the app.

---

### Step 4 — Endpoint `POST /auth/verify-code`

- [ ] **POST /auth/verify-code**

Accepts code. Hash the incoming code, SELECT by `code_hash` WHERE `used = false` AND `expires_at > now`. If not found — return 401. If found — in a single transaction mark `used = true` and create a session. The transaction is critical — without it a race condition can validate the same code twice. Return JWT access token + refresh token.

---

### Step 5 — Rate limiting on Cloudflare

- [ ] **Cloudflare rate limiting**

Cloudflare Dashboard → Security → WAF → Rate Limiting Rules. Two rules: `/auth/send-code` — max 3 requests per IP per 60 seconds. `/auth/verify-code` — max 10 attempts per IP per 5 minutes. This closes brute-force on the 9-digit code which has 900 million combinations anyway.

---

### Step 6 — Cleanup of expired records

- [ ] **Expired codes cleanup**

Simple server-side cron once per hour: `DELETE FROM auth_codes WHERE expires_at < now`. Alternatively use pg_cron directly in Postgres. Without this the table grows indefinitely.

---

### Step 7 — Flutter: deeplink handling and verification

- [ ] **Flutter: deeplink → verify flow**

App receives the deeplink, parses query parameter `code`, sends POST to `/auth/verify-code`, receives tokens, stores them in secure storage (flutter_secure_storage), redirects user to the home screen. If the server returns 401 — show an error screen with a "resend code" button.

---

### Step 8 — Remove Firebase

- [ ] **Remove Firebase (backend)** — delete firebase module, FirebaseAuthGuard, firebase-admin SDK dependency; clean up app.module.ts
- [ ] **Remove Firebase (mobile)** — remove firebase_auth, firebase_core dependencies, FlutterFire config files, flavor-specific google-services.json / GoogleService-Info.plist, firebase.json; update pubspec and build configs
- [ ] **Auth interceptor update** — wire Dio AuthInterceptor to new JWT access/refresh token storage; remove any Firebase ID token logic

---

## Completed

| Milestone | Date |
|-----------|------|
