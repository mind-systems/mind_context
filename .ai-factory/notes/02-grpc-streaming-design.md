# Socket.io → gRPC Streaming Design

**Date:** 2026-03-24
**Source:** research spike (roadmap phase 3.1)
**Full design doc:** mind_api/docs/grpc/streaming-design.md

## Key Findings

- The `/live` namespace maps to a **bidirectional gRPC stream** (`LiveSession`). Both sides produce messages independently: the client sends activity/presence commands; the server pushes `session:state` and `session:error` back at any time, including proactively on reconnect resume. Pure request/response would break this.
- The `/telemetry` namespace maps to a **bidirectional gRPC stream** (`StreamTelemetry`), not client-streaming, because the server sends a `data:ack` for every incoming batch (carrying `receivedCount`, `droppedCount`, and flow-control hints). Client-streaming cannot deliver per-message acks without closing the stream.
- `sync:changed` maps to **server-streaming** (`WatchChanges`). The client sends one subscription request; the server pushes debounced change-event batches whenever `CHANGE_EVENT_LOGGED` fires. No client → server messages after initial subscribe.
- The grace-timer logic in `GraceTimerManager` requires no structural change. Stream close (HTTP/2 reset) triggers `activityEngine.onDisconnect()` + `graceTimerManager.startTimer()`, identical to `handleDisconnect` today. A new stream opening re-triggers `graceTimerManager.cancelTimer()` + `activityEngine.resumeActivity()`, identical to `handleConnection` today.
- `StateStore.socketMap` (maps userId → Socket for broadcasting and single-connection enforcement) must be replaced by a `streamMap` that maps userId → `ServerWritableStream`. `presenceMap` and `activityMap` are transport-agnostic and require no change.

## Details

Three gRPC RPCs replace the two Socket.io namespaces:

**`LiveService.LiveSession`** — bidi stream. `LiveRequest` is a oneof covering all seven inbound events (`activity:start/end/stop/pause/resume`, `presence:background/foreground`). `LiveResponse` is a oneof covering `session:state` and `session:error`. The server-side handler replaces `LiveGateway` and drives the same `ActivityEngine`, `PresenceService`, and `GraceTimerManager` services.

**`TelemetryService.StreamTelemetry`** — bidi stream. `TelemetryData` carries `session_id`, `timestamp`, and serialised sample payload. `TelemetryAck` mirrors the current `data:ack` envelope. The server-side handler replaces `TelemetryGateway` and calls `StreamEngine.push()` unchanged.

**`SyncService.WatchChanges`** — server-streaming. `SyncNotifierService.flush()` replaces `socket.emit(SYNC_CHANGED, ...)` with `stream.write(ChangeEvent(...))`. The 300 ms debounce and event batching are unchanged.

Flutter `LiveSocketService` is replaced by a single gRPC service class managing three call objects (one per RPC). The `BehaviorSubject<SocketConnectionState>` and broadcast stream controllers remain; they are now driven by gRPC response stream close/error events. Reconnect on stream close must be implemented explicitly with exponential back-off (equivalent to the current `setReconnectionDelay(1000)` / max 30 s), since `package:grpc` does not auto-reconnect closed RPC calls.

## Open Questions

- How is the JWT delivered and refreshed on long-lived gRPC streams? (Socket.io passes it in `auth.token` at handshake; gRPC uses metadata headers.)
- Should `WatchChanges` replay missed events from the changelog on reconnect (requires `after_id` cursor support), or emit only the live debounce window as today?
- Multi-instance deployment: `streamMap` is in-process; a reconnecting client may land on a different instance and miss `sync:changed` pushes. Sticky sessions or Redis pub/sub needed.
- If per-batch acks are dropped in a future optimisation, `StreamTelemetry` could be downgraded to client-streaming — worth deciding before proto is locked.
- Rate limiting keyed on socket ID (`rateLimiterService.evict(client.id)`) must be rekeyed on userId or stream handle.
