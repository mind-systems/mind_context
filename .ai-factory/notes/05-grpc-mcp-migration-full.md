# gRPC Migration Design: mind_mcp

**Date:** 2026-03-24
**Status:** Draft
**Context:** Phase 5 work — HTTP controllers removed from mind_api after Phase 4; mind_mcp must switch from REST fetch() to gRPC.

---

## 1. Recommended Library Stack

**Use `@grpc/grpc-js` + `ts-proto`.**

Do not use the native `grpc` package (requires native bindings, incompatible with bundling). Do not use `@grpc/grpc-js` alone with runtime proto loading for this process.

| Package | Role |
|---|---|
| `@grpc/grpc-js` | gRPC channel, credentials, call execution (pure JS, no native bindings) |
| `ts-proto` | Build-time code generator — produces typed TypeScript stubs from `.proto` files |

`@grpc/proto-loader` is deliberately excluded from runtime dependencies. It is only needed as a build-time tool if you choose runtime loading — we do not.

### Why not runtime proto loading (`@grpc/proto-loader`)?

Runtime proto loading parses `.proto` files on every process start using a JS proto parser. For a stdio MCP server this matters: Claude Desktop spawns the process per session start. Parsing protos on every cold start adds 50–150ms of synchronous file I/O and parsing work before any tool call can be made. It also means shipping `.proto` files alongside the compiled `dist/` bundle, complicating the npm package layout (the package currently ships only `dist/`).

### Why ts-proto?

- Stubs are generated once at build time (`npm run proto:gen`) and compiled into `dist/` as normal TypeScript.
- Zero runtime proto parsing — the channel is ready immediately on process start.
- Full TypeScript types for request/response shapes, replacing the hand-maintained `types.ts` interfaces for any domain objects defined in proto.
- The generated code is standard TypeScript; no magic at runtime.

### Build-time dependency only

`ts-proto` and `protoc` (or `protoc-gen-ts_proto`) are `devDependencies`. Only `@grpc/grpc-js` is a runtime dependency.

---

## 2. Proto Loading Strategy: Compile-Time (ts-proto)

**Decision: compile-time generated stubs.**

The `.proto` source files live in `mind_api/proto/` (to be defined as part of the API migration). The MCP package adds a build step:

```
npm run proto:gen
  → protoc --plugin=protoc-gen-ts_proto=./node_modules/.bin/protoc-gen-ts_proto \
           --ts_proto_out=src/generated \
           --ts_proto_opt=outputServices=grpc-js,esModuleInterop=true \
           mind_api/proto/breath_sessions.proto
```

Generated stubs live in `src/generated/` and are checked in (or regenerated as part of CI). The `npm run build` step compiles them along with everything else.

**What gets generated:**
- `BreathSessionsServiceClient` class (typed gRPC client)
- Request/response message interfaces (`ListSessionsRequest`, `GetSessionResponse`, etc.)
- Enum types

---

## 3. PAT Authentication via gRPC Metadata

In REST, the current client sends `Authorization: Bearer <token>` as an HTTP header. In gRPC, this maps directly to a metadata key.

### Approach: CallCredentials factory attached to the channel

Rather than constructing a `Metadata` object manually in every call, attach the PAT as `CallCredentials` at channel creation time. This means every RPC automatically carries the token — matching how the current `request()` helper injects the header for every fetch call.

```typescript
import * as grpc from "@grpc/grpc-js";

function makePatCallCredentials(token: string): grpc.CallCredentials {
  return grpc.credentials.createFromMetadataGenerator(
    (_params, callback) => {
      const meta = new grpc.Metadata();
      meta.add("authorization", `Bearer ${token}`);
      callback(null, meta);
    }
  );
}
```

This credential is combined with channel credentials at construction:

```typescript
const channelCreds = grpc.credentials.combineChannelCredentials(
  tlsCreds,          // ChannelCredentials (insecure or TLS — see §5)
  makePatCallCredentials(token)  // CallCredentials
);
```

The server reads `metadata.get("authorization")[0]` and validates the PAT exactly as it did the HTTP `Authorization` header.

---

## 4. Error Mapping: gRPC Status → MCP Error String

gRPC errors surface as `ServiceError` objects (extends `Error`) with a numeric `code` property corresponding to `grpc.status` values.

### Utility function

```typescript
import * as grpc from "@grpc/grpc-js";
import type { ServiceError } from "@grpc/grpc-js";

export function grpcErrorToMcpError(err: ServiceError): string {
  switch (err.code) {
    case grpc.status.NOT_FOUND:
      return "Session not found";
    case grpc.status.UNAUTHENTICATED:
      return "Invalid or expired token";
    case grpc.status.PERMISSION_DENIED:
      return "Access denied";
    case grpc.status.INVALID_ARGUMENT:
      return `Invalid request: ${err.details}`;
    case grpc.status.ALREADY_EXISTS:
      return "Session already exists";
    case grpc.status.RESOURCE_EXHAUSTED:
      return "Rate limit exceeded";
    case grpc.status.UNAVAILABLE:
      return "Service temporarily unavailable — please retry";
    case grpc.status.INTERNAL:
      return "Internal server error";
    case grpc.status.UNIMPLEMENTED:
      return "Operation not supported";
    default:
      return `Unexpected error (gRPC code ${err.code}): ${err.details ?? err.message}`;
  }
}
```

### Status code mapping table

| gRPC status | Numeric | MCP error string |
|---|---|---|
| `NOT_FOUND` | 5 | Session not found |
| `UNAUTHENTICATED` | 16 | Invalid or expired token |
| `PERMISSION_DENIED` | 7 | Access denied |
| `INVALID_ARGUMENT` | 3 | Invalid request: `<details>` |
| `ALREADY_EXISTS` | 6 | Session already exists |
| `RESOURCE_EXHAUSTED` | 8 | Rate limit exceeded |
| `UNAVAILABLE` | 14 | Service temporarily unavailable — please retry |
| `INTERNAL` | 13 | Internal server error |
| `UNIMPLEMENTED` | 12 | Operation not supported |
| other | — | Unexpected error (gRPC code N): `<details>` |

The tool catch blocks replace `${err}` string interpolation with `grpcErrorToMcpError(err as ServiceError)`.

---

## 5. grpc-client.ts Design

This file replaces `src/api/client.ts`. It exports the same four functions so that all tool files require zero changes beyond updating the import path (or keeping the same path and renaming the file).

### Environment variables

| Variable | Purpose | Example |
|---|---|---|
| `MIND_GRPC_URL` | Host and port of the gRPC server | `localhost:5000` |
| `MIND_PAT_TOKEN` | Personal Access Token | `pat_abc123` |
| `MIND_GRPC_TLS` | Set to `true` to use TLS; omit for insecure | `true` |

`MIND_API_URL` is retired.

### Channel creation and credentials

```typescript
const url = process.env.MIND_GRPC_URL;
const token = process.env.MIND_PAT_TOKEN;

if (!url || !token) {
  throw new Error("MIND_GRPC_URL and MIND_PAT_TOKEN must be set");
}

const useTls = process.env.MIND_GRPC_TLS === "true";

const baseCreds = useTls
  ? grpc.credentials.createSsl()          // system CA bundle
  : grpc.credentials.createInsecure();

const channelCreds = grpc.credentials.combineChannelCredentials(
  baseCreds,
  makePatCallCredentials(token)
);

const client = new BreathSessionsServiceClient(url, channelCreds);
```

For local development (`MIND_GRPC_TLS` unset), the channel is insecure. For production, set `MIND_GRPC_TLS=true`; `createSsl()` uses the system certificate store and needs no additional configuration for standard certs. Self-signed certs require passing the CA buffer to `createSsl(rootCerts)`.

### Exported functions (matching current client.ts surface)

```typescript
export async function fetchSessions(
  page?: number,
  pageSize?: number,
): Promise<BreathSessionListResponse>;

export async function fetchSession(id: string): Promise<BreathSession>;

export async function patchSession(
  id: string,
  data: Partial<BreathSession>,
): Promise<BreathSession>;

export async function createSession(
  data: CreateBreathSessionPayload,
): Promise<BreathSession>;
```

Each function wraps the generated stub call in a Promise (the generated `grpc-js` stubs use Node callbacks by default unless `ts-proto` is configured with `outputServices=nice-grpc` — see Open Questions). Error handling translates `ServiceError` via `grpcErrorToMcpError` and re-throws as a plain `Error` so existing tool catch blocks continue working unchanged.

### Stub call wrapper pattern

```typescript
function callUnary<Req, Res>(
  fn: (req: Req, meta: grpc.Metadata, cb: (err: ServiceError | null, res: Res) => void) => void,
  req: Req,
): Promise<Res> {
  return new Promise((resolve, reject) => {
    fn(req, new grpc.Metadata(), (err, res) => {
      if (err) reject(new Error(grpcErrorToMcpError(err)));
      else resolve(res!);
    });
  });
}
```

With `combineChannelCredentials` the per-call `Metadata` argument is empty; the PAT is injected automatically by the `CallCredentials` factory.

---

## 6. Migration Steps Per Tool

All tool files currently import from `../api/client.js`. The migration path is:

1. **No changes needed in tool files** if `src/api/client.ts` is replaced in place with the gRPC implementation and exports the same four function signatures.
2. Only the `types.ts` interfaces that overlap with proto-generated types may need updating — if `ts-proto` generates `BreathSession`, `BreathSessionListResponse`, and `CreateBreathSessionPayload` from the proto definitions, those hand-maintained types become redundant and should be removed in favour of the generated ones.

Per-tool impact:

| Tool file | Uses | Impact |
|---|---|---|
| `listSessions.ts` | `fetchSessions` | None — same signature |
| `getSession.ts` | `fetchSession` | None — same signature |
| `createSession.ts` | `createSession` | None if proto message matches current payload shape |
| `classifySession.ts` | `fetchSession` | None |
| `classifyAll.ts` | `fetchSessions` | None — pagination parameters unchanged |
| `setTimeOfDay.ts` | `patchSession` | May need review: `PATCH` semantics become an `UpdateSession` RPC; verify field mask or dedicated `SetTimeOfDay` RPC |

`setTimeOfDay.ts` deserves the most attention. A REST `PATCH` with a partial body maps less cleanly to gRPC. The API should expose either a `UpdateSession` RPC with a `FieldMask` or a dedicated `SetTimeOfDay` RPC. The MCP function signature stays the same either way.

---

## 7. Open Questions

1. ~~**nice-grpc vs raw callback stubs.**~~ **Resolved: raw callback stubs (`outputServices=grpc-js`).** `nice-grpc` would eliminate the `callUnary` wrapper but adds two runtime packages (`nice-grpc`, `nice-grpc-common`) to a stdio process. The wrapper is 10 lines written once. PAT injection is already handled cleanly via `CallCredentials` factory on the channel (§3) — no per-call middleware needed. Net cost of `nice-grpc`: more dependencies, no meaningful gain for this use case.

2. ~~**Proto file ownership.**~~ **Resolved.** `mind_api/proto/` is the single source of truth. No other project may modify `.proto` files. Any contract change starts in `mind_api/proto/`, then each consumer (`mind_mcp`, `mind_mobile`) copies the updated files and regenerates stubs. Symlinks are rejected (fragile on Windows, breaks when repos are cloned independently). Copy-on-change is explicit and safe. Rule documented in all CLAUDE.md files.

3. ~~**Self-signed certs in staging.**~~ **Not applicable.** Both prod (`api.mind-awake.life`) and dev (`dev-api.mind-awake.life`) use standard public certs — `createSsl()` with the system CA bundle works for both. Local development uses `createInsecure()` (no `MIND_GRPC_TLS` set) — no TLS needed on localhost. No custom CA handling required.

4. ~~**SetTimeOfDay vs UpdateSession RPC.**~~ **Resolved.** `timeOfDay` is part of `UpdateBreathSessionDto` — it belongs to `UpdateSession` RPC, not `UpdateSessionSettings` (which only covers `starred`). `setTimeOfDay.ts` calls `UpdateSession({ id, time_of_day })`. No dedicated RPC needed.

5. ~~**Streaming for classifyAll.**~~ **Out of scope.** Sequential paginated calls work fine for this use case — even 500 sessions is ~10 calls at ~10 ms each, imperceptible for an MCP tool. A `StreamSessions` RPC would require a new proto definition, new controller, and stream client code for negligible gain.

6. ~~**Channel reuse.**~~ **Resolved.** Single `BreathSessionsServiceClient` instance at module load time is correct — the process lives for the duration of the Claude Desktop session, all tool calls share one channel. This mirrors the current fetch-based client where `MIND_API_URL` and token are module-level constants. No per-call channel creation.
