---
name: aif-review-mobile
description: Code review for mind_mobile (Flutter + Riverpod + Drift). Project-specific checklist covering StreamSubscription leaks, domain layer purity, sealed class completeness, Riverpod architecture, and API sync. Called automatically by aif-review when mobile changes are detected, or invoke directly.
argument-hint: "[PR number or empty]"
allowed-tools: Bash(git *) Bash(gh *) Read Glob Grep
---

# Code Review — mind_mobile (Flutter)

Perform thorough code review of mobile changes, focusing on correctness, Flutter conventions, Riverpod architecture, and project-specific patterns.

## Behavior

### Without Arguments (Review Staged Changes)

1. Run `git diff --cached` to get staged changes
2. If nothing staged, run `git diff` for unstaged changes
3. Analyze each changed file under `mind_mobile/`

### With PR Number/URL

1. Use `gh pr view <number> --json title,body` to get PR details and context
2. Use `gh pr diff <number>` to get the full diff
3. Review all `mind_mobile/` changes in the PR

## Review Checklist

### Correctness
- [ ] Logic errors or bugs
- [ ] Edge cases handling
- [ ] Null safety — proper `?` and `!` usage, no forced unwrap without guard
- [ ] Error handling completeness

### Security
- [ ] Sensitive data not stored unencrypted in Drift or shared preferences
- [ ] Auth tokens not logged or exposed in error messages

### Performance
- [ ] Every `StreamSubscription` field cancelled in `dispose()` — missing cancel causes memory leaks and ghost events
- [ ] Unnecessary widget rebuilds (watching too broad a provider)
- [ ] Large payloads cached unnecessarily in Drift

### Flutter / Dart (this project)
- [ ] Every `StreamSubscription` field has `?.cancel()` call in `dispose()`
- [ ] Domain layer classes (`Notifier`, `Repository`) have **no** `flutter/*` or `flutter_riverpod` imports — framework must not leak into domain
- [ ] `sealed class` switch statements and socket status string handling cover **all** cases — no missing status strings that leave state stuck
- [ ] Side effects (socket subscriptions, session start, network calls) triggered by explicit user actions, not `initState`/constructor lifecycle hooks
- [ ] New `Notifier` or `Service` classes added to `App.dart` in correct initialization order: DB → API → Repository → Notifier
- [ ] When a `Notifier` listens to another `Notifier` (e.g. `UserNotifier`), the stream is injected as `Stream<T>` rather than the concrete class — keeps unit tests feasible without constructing heavyweight dependencies

### Architecture (Riverpod + Domain)
- [ ] No business logic in widgets — widgets call notifier methods only
- [ ] Repositories expose typed domain models, not raw API DTOs
- [ ] Drift schema changes have a corresponding migration added

### API Sync (if touching models or API layer)
- [ ] Dart model fields match the current NestJS response shape
- [ ] Drift cache schema updated if response shape changed
- [ ] `AuthInterceptor.dart` updated if auth flow changed
- [ ] `lib/Core/Api/` Dio client updated if new endpoints added

### Testing
- [ ] Test coverage for new notifier state transitions
- [ ] StreamSubscription lifecycle verified (subscribe → dispose → no further events)
- [ ] Edge cases tested

## Output Format

```markdown
## Code Review — mind_mobile

**Files Reviewed:** [count]
**Risk Level:** 🟢 Low / 🟡 Medium / 🔴 High

### Critical Issues
[Must be fixed before merge]

### Suggestions
[Nice to have improvements]

### Questions
[Clarifications needed]

### Positive Notes
[Good patterns observed]
```

## Review Style

- Be constructive, not critical
- Explain the "why" behind suggestions
- Provide code examples when helpful
- Acknowledge good code
- Prioritize feedback by importance
- Ask questions instead of making assumptions

## Examples

**User:** `/aif-review-mobile`
Review staged mobile changes in current repository.

**User:** `/aif-review-mobile 123`
Review mobile changes in PR #123 using GitHub CLI.

**User:** `/aif-review-mobile https://github.com/org/repo/pull/123`
Review mobile changes from PR URL.

## Integration

If GitHub MCP is configured, can:
- Post review comments directly to PR
- Request changes or approve
- Add labels based on review outcome

> **Tip:** Context is heavy after code review. Consider `/clear` or `/compact` before continuing with other tasks.
