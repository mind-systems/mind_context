# gRPC Streaming Design: Socket.io Replacement

**Date:** 2026-03-24
**Status:** Design spike

---

## Executive Summary

The Mind API currently exposes two Socket.io namespaces for real-time communication: `/live` (activity lifecycle + presence) and `/telemetry` (biometric sample batches). A third real-time channel — `sync:changed` — pushes data-change notifications over the `/live` socket from `SyncNotifierService`. This document maps each of these channels to an appropriate gRPC streaming pattern and addresses the key engineering challenges: reconnection, grace-period logic, in-memory state migration, and Flutter client implications.

The central insight is that the existing architecture already cleanly separates concerns in a way that maps naturally onto three distinct gRPC RPCs, each with a different streaming pattern.

---

## Event Mapping Table

| Socket.io channel | Direction | Events / messages | gRPC pattern | Rationale |
|---|---|---|---|---|
| `/live` namespace | Bidirectional | Client sends: `activity:start`, `activity:end`, `activity:stop`, `activity:pause`, `activity:resume`, `presence:background`, `presence:foreground`. Server sends: `session:state`, `session:error` | **Bidirectional streaming** (`LiveSession`) | Both sides produce messages independently. The server must be able to push `session:state` at any point (e.g., after reconnect resume), not only in response to the immediately preceding client message. Pure request/response is insufficient. |
| `/telemetry` namespace | Client → Server with per-message ack | Client sends: `data:stream` (one message per sample batch). Server sends: `data:ack` (per batch, carries `receivedCount`, `droppedCount`) | **Bidirectional streaming** (`StreamTelemetry`) | Although the flow is mostly client-to-server, the server sends a `data:ack` for every incoming batch. Client-streaming closes the response stream only at RPC end — incompatible with per-message acks needed for flow control and dropped-sample signalling. Bidi streaming allows the server to ack each batch immediately. |
| `sync:changed` push | Server → Client | Server pushes batched change events (debounced 300 ms) after any `CHANGE_EVENT_LOGGED` emission | **Server-streaming** (`WatchChanges`) | The client sends a single subscription request and then only listens. No messages travel client → server after the initial request. |

---

## Proto Sketch

```proto
syntax = "proto3";
package mind.realtime.v1;

// ── live.proto ──────────────────────────────────────────────────────────────

service LiveService {
  // Bidirectional: client sends activity/presence commands; server pushes
  // session state and errors back. One stream per connected user.
  rpc LiveSession(stream LiveRequest) returns (stream LiveResponse);
}

message LiveRequest {
  oneof command {
    ActivityStartCmd  activity_start   = 1;
    ActivityEndCmd    activity_end     = 2;
    ActivityStopCmd   activity_stop    = 3;
    ActivityPauseCmd  activity_pause   = 4;
    ActivityResumeCmd activity_resume  = 5;
    PresenceCmd       presence         = 6;
  }
}

message ActivityStartCmd {
  string activity_type     = 1;
  string activity_ref_type = 2;  // optional
  string activity_ref_id   = 3;  // optional
}

// ActivityEndCmd, ActivityStopCmd, ActivityPauseCmd, ActivityResumeCmd
// carry no payload — the stream context identifies the user.
message ActivityEndCmd    {}
message ActivityStopCmd   {}
message ActivityPauseCmd  {}
message ActivityResumeCmd {}

enum PresenceStatus { FOREGROUND = 0; BACKGROUND = 1; }
message PresenceCmd { PresenceStatus status = 1; }

message LiveResponse {
  oneof event {
    SessionStateEvent session_state = 1;
    SessionErrorEvent session_error = 2;
  }
}

message SessionStateEvent {
  string live_session_id = 1;
  string status          = 2;  // ACTIVE | COMPLETED | INTERRUPTED | RESUMED
  bool   is_paused       = 3;
}

message SessionErrorEvent {
  string code      = 1;
  string message   = 2;
  int64  timestamp = 3;
}


// ── telemetry.proto ──────────────────────────────────────────────────────────

service TelemetryService {
  // Bidirectional: client streams sample batches; server acks each batch.
  rpc StreamTelemetry(stream TelemetryData) returns (stream TelemetryAck);
}

message TelemetryData {
  string session_id = 1;
  int64  timestamp  = 2;
  bytes  data       = 3;  // serialised DataStreamDto payload
}

message TelemetryAck {
  string session_id         = 1;
  int64  received_count     = 2;
  int32  max_samples_per_sec = 3;
  int64  timestamp          = 4;
  optional int64 dropped_count = 5;
  optional string error     = 6;  // SESSION_PAUSED | SESSION_MISMATCH | NO_SESSION
}


// ── sync.proto ───────────────────────────────────────────────────────────────

service SyncService {
  // Server-streaming: client subscribes once; server pushes change batches.
  rpc WatchChanges(WatchChangesRequest) returns (stream ChangeEvent);
}

message WatchChangesRequest {
  // Optional cursor: server may skip events the client already holds.
  optional int64 after_id = 1;
}

message ChangeEvent {
  repeated ChangeRecord events = 1;
}

message ChangeRecord {
  int64  id     = 1;
  string entity = 2;
  string ref_id = 3;
  string action = 4;
}
```

---

## Reconnection and Grace-Timer Strategy

### The problem

Socket.io reconnect is transparent to application code: when the client re-establishes the connection, the same socket ID and namespace context are restored. `LiveGateway.handleConnection` detects an existing `activityMap` entry for the user, cancels the grace timer, and calls `activityEngine.resumeActivity()` — all within the connection callback.

With gRPC over HTTP/2, a network drop closes the HTTP/2 stream immediately. There is no session-layer concept of reconnect; the client must open a new RPC call. From the server's perspective the old call's context is gone as soon as the stream closes, which is equivalent to a socket disconnect.

### Proposed strategy

1. **Stream close triggers the same logic as socket disconnect.** The gRPC server interceptor (or the stream's `on('close')` handler) calls `activityEngine.onDisconnect(userId)` and starts the grace timer, exactly as `handleDisconnect` does today.

2. **New stream open triggers the same logic as socket reconnect.** On the first message of a new `LiveSession` call (or on a dedicated `ReconnectCmd` message type), the server checks `activityMap.has(userId)`. If a pending session is found, it cancels the grace timer and calls `activityEngine.resumeActivity()`, then pushes `session:state { status: RESUMED }` down the new stream.

3. **Grace timer duration is unchanged.** `WS_RECONNECT_GRACE_MS` (default 30 s) remains the authority. The rename to something transport-neutral (e.g. `RECONNECT_GRACE_MS`) is cosmetic but worthwhile.

4. **Single-connection enforcement.** Currently `handleConnection` evicts the previous socket for the same `userId`. With gRPC the equivalent is: on opening a new `LiveSession` stream, the server must terminate any existing server-side stream handle for that user before registering the new one (see State-Store Migration below).

### What happens to `activityMap` during stream reset

`activityMap` is in-memory and lives on the NestJS process — it is unaffected by stream closure. The entry persists exactly as it does after a socket disconnect. The grace timer fires only if the client does not re-open a stream within `RECONNECT_GRACE_MS`. This behaviour is identical to the current implementation; no changes to `ActivityEngine` are required.

---

## State-Store Migration

### `socketMap`

`socketMap` currently serves two purposes:

1. **Single-connection enforcement** — `handleConnection` checks and evicts the previous socket for a user.
2. **Push target for `SyncNotifierService`** — `sync-notifier.service.ts` calls `socketMap.get(userId)` to obtain the `Socket` object and emit `sync:changed` on it.

With gRPC streaming, the Socket object is replaced by a **server-side writable stream handle** (in Node.js gRPC terms, a `ServerDuplexStream` or `ServerWritableStream`). `socketMap` should be replaced by a typed map:

```typescript
// Replaces StateStore.socketMap
readonly streamMap = new Map<string, ServerWritableStream<any, any>>();
```

`SyncNotifierService.flush()` calls `stream.write(changeEvent)` instead of `socket.emit(SYNC_CHANGED, ...)`. The debounce logic and event batching are unchanged.

### `presenceMap` and `activityMap`

These maps are keyed by `userId` and store application-level state, not transport handles. They require no structural change. The only change is that the write operations (set/delete/read) are now triggered by gRPC stream events rather than Socket.io connection callbacks.

---

## Flutter Client Implications

### `package:grpc` bidi stream model

In the `grpc` Dart package, a bidirectional call returns a `ResponseStream<LiveResponse>`. The client writes to the call object with `call.sendMessage(LiveRequest(...))` and listens to responses via `call.toList()` or by subscribing to the response stream. The call object encapsulates both the send and receive channels.

### Replacing `LiveSocketService`

The current `LiveSocketService` maintains two Socket.io sockets (`_liveSocket`, `_telemetrySocket`) and a separate listener for `sync:changed` on `_liveSocket`. Under gRPC, this becomes three call objects:

- `_liveCall` — `ClientBidirectionalStreamingCall<LiveRequest, LiveResponse>` — sends activity/presence commands; receives session state and errors.
- `_telemetryCall` — `ClientBidirectionalStreamingCall<TelemetryData, TelemetryAck>` — sends sample batches; receives acks.
- `_syncCall` — `ResponseStream<ChangeEvent>` (server-streaming) — no sends; subscribes to change events.

The `BehaviorSubject<SocketConnectionState>` and the stream controllers (`_sessionStateController`, `_dataAckController`, `_syncChangedController`) remain structurally intact — they are driven by gRPC response stream events instead of Socket.io event listeners.

### Reconnection in Flutter

`package:grpc` does not reconnect a closed stream automatically. The channel-level keepalive (`channelOptions: ChannelOptions(keepAlive: ...)`) keeps the underlying HTTP/2 connection alive but does not reopen closed RPCs. The `LiveSocketService` replacement must implement reconnect logic explicitly:

1. Subscribe to gRPC call completion (the `ResponseStream` will close with a `GrpcError` on disconnect).
2. On close with a retryable status (e.g., `unavailable`, `internal`), wait with exponential back-off (matching the current `setReconnectionDelay(1000)` / `setReconnectionDelayMax(30000)`) and re-open the RPC call.
3. On re-open, emit `SocketConnectionState.connecting` → `connected` (or `disconnected` if retries exhausted), matching the existing state machine.

The current double-deduplication guard (`_isConnecting` flag + null check after await) has a direct analogue: a single `_isReconnecting` boolean guards the reconnect path.

---

## Open Questions

1. ~~**Authentication token delivery.**~~ **Resolved.** gRPC metadata `Authorization` header is validated once when the stream is opened — identical to how `WsAuthMiddleware` authenticates only at `connect`. Token refresh is a non-issue: JWTs in this system do not expire on a timer; validity is determined by presence in `user_sessions` (SHA-256 hash lookup). A session is invalidated only by explicit `revoke()` (logout). The current Socket.io implementation does not re-validate mid-connection either (`WsAuthGuard` only asserts `userId` is present in `client.data`). The gRPC server interceptor follows the same pattern: validate on stream open, store `userId` in stream context, no periodic re-checks.

2. ~~**Rate limiting.**~~ **Resolved.** `RateLimiterService` is a generic `Map<key, WindowEntry>`. In Socket.io the key is `client.id`; in gRPC it becomes `userId` — a straight substitution. `consume(userId, limit, windowMs)` and `evict(userId)` on stream close. This is actually an improvement: per-userId rate limiting is stricter than per-connection (a user on two devices can no longer bypass the limit). The Map does not leak even without `evict` because entries reset on the next window tick; `evict` is just a memory optimisation.

3. ~~**Payload size enforcement.**~~ **Resolved.** gRPC handles this at the protocol level: `@grpc/grpc-js` enforces `maxReceiveMessageLength` (default 4 MB) on the channel, and protobuf schema typing prevents arbitrary blobs. No equivalent of `WsPayloadSizeGuard` is needed. Optionally set `maxReceiveMessageLength` explicitly in the server `ChannelOptions` for visibility, but the default is sufficient.

4. ~~**Telemetry bidi vs client-streaming trade-off.**~~ **Resolved: bidi.** `TelemetryGateway` emits `DATA_ACK` on every batch with `droppedCount` — this is a real-time backpressure signal, not just diagnostics. If the buffer cap is hit, the client needs to know immediately to reduce send rate. Client-streaming would only deliver this at stream close, making the client blind during the session. Bidi is the correct pattern and maps 1:1 to the current Socket.io behaviour.

5. ~~**`sync:changed` cursor semantics.**~~ **Resolved.** `ChangeLogService.getChanges(userId, afterId, limit)` already supports replay from the DB — the infrastructure exists. `WatchChanges` server-streaming RPC should work in two phases: (1) replay all missed events from DB using `after_id` from the request (paginated via `hasMore`), then (2) switch to live push by subscribing to `CHANGE_EVENT_LOGGED` for that userId. On reconnect the client passes its last known cursor and gets a gapless stream. No new architecture required.

6. ~~**Multi-instance deployment.**~~ **Out of scope.** The API currently runs as a single instance (no `replicas` in docker-compose.prod.yml). The in-process `streamMap` has the same limitation as the current `socketMap` — this is a pre-existing concern not introduced by the gRPC migration. Horizontal scaling with pub/sub fan-out (Redis) is a separate future initiative, tracked in roadmap Phase 6.

7. ~~**`connectedAt` metrics.**~~ **Resolved.** Trivial port: replace `Map<socketId, timestamp>` with `Map<userId, timestamp>` (or stream handle). Record `Date.now()` on stream open, compute diff on stream close, log `connectedDurationMs`. No design decision needed.
