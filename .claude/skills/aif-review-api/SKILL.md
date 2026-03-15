---
name: aif-review-api
description: Code review for mind_api (NestJS + TypeORM + PostgreSQL + Socket.io). Project-specific checklist covering NestJS conventions, WebSocket patterns, TypeORM migrations, ConfigService rules, and lifecycle completeness. Called automatically by aif-review when API changes are detected, or invoke directly.
argument-hint: "[PR number or empty]"
allowed-tools: Bash(git *) Bash(gh *) Read Glob Grep
---

# Code Review — mind_api (NestJS)

Perform thorough code review of API changes, focusing on correctness, security, NestJS conventions, and project-specific patterns.

## Behavior

### Without Arguments (Review Staged Changes)

1. Run `git diff --cached` to get staged changes
2. If nothing staged, run `git diff` for unstaged changes
3. Analyze each changed file under `mind_api/`

### With PR Number/URL

1. Use `gh pr view <number> --json title,body` to get PR details and context
2. Use `gh pr diff <number>` to get the full diff
3. Review all `mind_api/` changes in the PR

## Review Checklist

### Correctness
- [ ] Logic errors or bugs
- [ ] Edge cases handling
- [ ] Null/undefined checks where fields are optional
- [ ] Error handling completeness (exceptions caught and transformed to HTTP responses)
- [ ] Type safety — no `as unknown as ...` casts, use proper interface extension

### Security
- [ ] SQL injection (raw queries use parameters, never string concatenation)
- [ ] XSS vulnerabilities in any templated output
- [ ] Input validation — `class-validator` decorators present on all DTOs
- [ ] Auth guards applied to new endpoints
- [ ] Sensitive data not logged or exposed in response bodies
- [ ] CSRF protection on mutating endpoints

### Performance
- [ ] N+1 query problems (TypeORM relations loaded without `relations:[]` or `QueryBuilder.leftJoinAndSelect`)
- [ ] Missing database indexes for newly filtered or sorted columns
- [ ] Large payload sizes

### NestJS Conventions
- [ ] No direct `process.env` access in services — all config via injected `ConfigService`, read in constructor and stored as `private readonly` fields
- [ ] Services implementing `OnApplicationBootstrap` that allocate resources (timers, intervals, buffers, connections) also implement `OnApplicationShutdown` with proper cleanup
- [ ] New environment-driven constants extracted via `ConfigService`, not hardcoded in service files
- [ ] No unsafe type casts (`as unknown as ...`) — extend interfaces or use type guards instead
- [ ] Field names in socket/API emissions are unambiguous and consistent with the rest of the protocol

### WebSocket (RealtimeModule)
- [ ] Services with timers/intervals implement both `OnApplicationBootstrap` AND `OnApplicationShutdown`
- [ ] No hardcoded WS constants — all tunable values (flush interval, rate limits, max payload, min duration) read from `ConfigService`
- [ ] State interface changes: all test fixtures constructing that type updated in the same commit (grep for interface name in spec files)
- [ ] Error emissions use consistent format: `{ code, message, timestamp }`
- [ ] Logging uses `new Logger(ClassName.name)`, not `console.log`

### TypeORM & Migrations
- [ ] Migrations generated via CLI (`npx typeorm migration:create`), not hand-crafted timestamps
- [ ] New entity fields have correct column types and `nullable`/`default` specified
- [ ] Relations defined on both sides if bidirectional

### Testing
- [ ] Test coverage for new logic paths
- [ ] When interface/entity fields are added: all test fixtures that construct that type are updated in the same commit
- [ ] `ConfigService` mocked (not `process.env`) in specs

## Output Format

```markdown
## Code Review — mind_api

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

**User:** `/aif-review-api`
Review staged API changes in current repository.

**User:** `/aif-review-api 123`
Review API changes in PR #123 using GitHub CLI.

**User:** `/aif-review-api https://github.com/org/repo/pull/123`
Review API changes from PR URL.

## Integration

If GitHub MCP is configured, can:
- Post review comments directly to PR
- Request changes or approve
- Add labels based on review outcome

> **Tip:** Context is heavy after code review. Consider `/clear` or `/compact` before continuing with other tasks.
