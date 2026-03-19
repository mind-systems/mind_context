# Logging Guidelines

**Keep logs minimal.** Only log errors and key state changes. Do not add verbose entry/exit logging or debug noise.

## What to log

1. **Errors** — always log with context (relevant IDs, input that caused it)
2. **Key state changes** — creation, deletion, status transitions (e.g. "session started", "payment completed")
3. **External call failures** — failed API requests, DB errors, timeout

## What NOT to log

- Function entry/exit
- Parameter values on every call
- Successful validation
- Happy-path flow ("processing started", "step 1 done", "step 2 done")
- Return values

## Example

```typescript
// Good: error with context
catch (error) {
  this.logger.error(`Failed to process order ${order.id}: ${error.message}`);
  throw error;
}

// Good: key state change
this.logger.log(`Session ${sessionId} completed, duration: ${duration}s`);

// Bad: verbose noise
this.logger.log(`[processOrder] START ${JSON.stringify(order)}`);
this.logger.log(`[processOrder] Validation passed`);
this.logger.log(`[processOrder] Calling payment service...`);
this.logger.log(`[processOrder] Payment returned ${JSON.stringify(result)}`);
```

## NEVER log sensitive data

This is a hard rule — violations are security incidents, not style issues.

**Never log:**
- JWT tokens, PAT tokens, refresh tokens, any auth tokens
- Email addresses, phone numbers, user PII
- Passwords, OTP codes, API keys, secrets
- Full request/response bodies (may contain any of the above)
- Authorization headers

**Instead:** log opaque identifiers (userId, sessionId) that can be used to trace without exposing sensitive data.

## Rules

- Use the project's existing logger (NestJS Logger, Flutter logger, etc.) — never raw `console.log`
- Use appropriate levels: `error` for failures, `warn` for recoverable issues, `log`/`info` for key events
- Include identifiers (userId, sessionId, etc.) for traceability
