# Flutter gRPC ClientInterceptor — Auth & Error Handling

**Date:** 2026-03-24
**Source:** research spike (roadmap phase 2.4)

## Key Findings

- Metadata injection in `package:grpc` is done by implementing `ClientInterceptor` and overriding `interceptUnary` / `interceptStreaming`; the interceptor receives a `CallOptions` and must merge new metadata into it before calling through.
- UNAUTHENTICATED errors surface as a `GrpcError` thrown by the RPC future (or stream); there is no dedicated error hook — you wrap the call in a try/catch inside the interceptor.
- Token reads from `FlutterSecureStorage` are async; `ClientInterceptor` methods are async (`Future`-returning), so `await _storage.read(key: 'jwt_token')` is safe and the correct approach.
- Interceptors are passed to a generated stub constructor via the `interceptors` named parameter: `MyServiceClient(channel, interceptors: [authInterceptor])`.

## Details

### ClientInterceptor API

`package:grpc` exposes the abstract class `ClientInterceptor` in `package:grpc/grpc.dart`. It has two methods to override:

```dart
abstract class ClientInterceptor {
  ResponseFuture<R> interceptUnary<Q, R>(
    ClientMethod<Q, R> method,
    Q request,
    CallOptions options,
    ClientUnaryInvoker<Q, R> invoker,
  );

  ResponseStream<R> interceptStreaming<Q, R>(
    ClientMethod<Q, R> method,
    Stream<Q> requests,
    CallOptions options,
    ClientStreamingInvoker<Q, R> invoker,
  );
}
```

- `method` — descriptor for the RPC (path, request/response serializers).
- `request` / `requests` — the outgoing message(s).
- `options` — a `CallOptions` instance carrying per-call metadata, timeout, and compression settings.
- `invoker` — the next handler in the chain (either the next interceptor or the actual channel send). Must be called to complete the RPC.

### Metadata Injection

`CallOptions` is immutable. To add metadata, merge a new `CallOptions` carrying your key/value pairs:

```dart
@override
ResponseFuture<R> interceptUnary<Q, R>(
  ClientMethod<Q, R> method,
  Q request,
  CallOptions options,
  ClientUnaryInvoker<Q, R> invoker,
) async {
  final token = await _storage.read(key: 'jwt_token');
  final merged = options.mergedWith(
    CallOptions(metadata: {
      if (token != null) 'authorization': 'Bearer $token',
    }),
  );
  return invoker(method, request, merged);
}
```

`CallOptions.mergedWith()` returns a new `CallOptions` that combines the original metadata with the supplied overrides, so existing caller-provided headers are preserved.

The metadata key must be lowercase `'authorization'` — gRPC metadata keys are case-insensitive on the wire but the Dart client sends them as-is; the NestJS/grpc server expects lowercase per the HTTP/2 spec.

### UNAUTHENTICATED Error Interception

There is no dedicated error callback in `ClientInterceptor`. You intercept errors by wrapping the `invoker` call:

```dart
@override
ResponseFuture<R> interceptUnary<Q, R>(
  ClientMethod<Q, R> method,
  Q request,
  CallOptions options,
  ClientUnaryInvoker<Q, R> invoker,
) async {
  final merged = await _buildOptions(options);
  try {
    return await invoker(method, request, merged);
  } on GrpcError catch (e) {
    if (e.code == StatusCode.unauthenticated) {
      _logoutNotifier.triggerLogout();
    }
    rethrow;
  }
}
```

`GrpcError` is in `package:grpc/grpc.dart`. `StatusCode.unauthenticated` equals gRPC status 16, which corresponds to HTTP 401.

For streaming RPCs the pattern is the same but you wrap the returned `ResponseStream`. Because `ResponseStream` is a `Stream`, you can transform it with `.handleError`:

```dart
@override
ResponseStream<R> interceptStreaming<Q, R>(
  ClientMethod<Q, R> method,
  Stream<Q> requests,
  CallOptions options,
  ClientStreamingInvoker<Q, R> invoker,
) {
  // Note: cannot await here (return type is ResponseStream, not Future).
  // Token must be read synchronously or the options built ahead of time.
  // Prefer the async unary pattern; streaming auth is handled separately.
  final stream = invoker(method, requests, options);
  return stream..handleError((Object e) {
    if (e is GrpcError && e.code == StatusCode.unauthenticated) {
      _logoutNotifier.triggerLogout();
    }
  });
}
```

**Important limitation for streaming:** `interceptStreaming` is not `async` because it must return a `ResponseStream` synchronously. This means `await _storage.read(...)` cannot be called directly inside it. Two workarounds:
1. Use a synchronously-available token cache (e.g., a field updated on each unary call or on login).
2. For the initial connection, require the channel to already have the token before opening a stream (the typical pattern — streaming is only started after at least one successful unary auth call).

### Token Retrieval Strategy

**Recommended: read directly from `FlutterSecureStorage` in `interceptUnary`.**

- The Dio `AuthInterceptor` already does this: `await _storage.read(key: _tokenKey)`. The same pattern is correct for the gRPC interceptor.
- Do NOT read from `UserNotifier.currentState` — `UserNotifier` is a domain-layer object and should not be injected into a transport-layer interceptor. This would create an upward dependency.
- Do NOT cache the token as a field on the interceptor — the cached value could become stale after a token rotation or re-login. Re-reading from secure storage on every call is cheap (in-process keychain read on iOS, EncryptedSharedPreferences on Android).
- The storage key is `'jwt_token'`, consistent across `AuthInterceptor`, `HttpClient`, and `UserRepository`.

### GrpcAuthInterceptor Design

```dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:grpc/grpc.dart';
import 'package:mind/User/LogoutNotifier.dart';

class GrpcAuthInterceptor implements ClientInterceptor {
  static const String _tokenKey = 'jwt_token';

  final FlutterSecureStorage _storage;
  final LogoutNotifier _logoutNotifier;

  GrpcAuthInterceptor({
    required FlutterSecureStorage storage,
    required LogoutNotifier logoutNotifier,
  })  : _storage = storage,
        _logoutNotifier = logoutNotifier;

  Future<CallOptions> _buildOptions(CallOptions options) async {
    final token = await _storage.read(key: _tokenKey);
    if (token == null) return options;
    return options.mergedWith(
      CallOptions(metadata: {'authorization': 'Bearer $token'}),
    );
  }

  void _handleError(Object e) {
    if (e is GrpcError && e.code == StatusCode.unauthenticated) {
      _logoutNotifier.triggerLogout();
    }
  }

  @override
  ResponseFuture<R> interceptUnary<Q, R>(
    ClientMethod<Q, R> method,
    Q request,
    CallOptions options,
    ClientUnaryInvoker<Q, R> invoker,
  ) async {
    final merged = await _buildOptions(options);
    try {
      return await invoker(method, request, merged);
    } on GrpcError catch (e) {
      _handleError(e);
      rethrow;
    }
  }

  @override
  ResponseStream<R> interceptStreaming<Q, R>(
    ClientMethod<Q, R> method,
    Stream<Q> requests,
    CallOptions options,
    ClientStreamingInvoker<Q, R> invoker,
  ) {
    // Cannot await here; token must be pre-fetched or streamed calls
    // must only be opened after unary auth has already succeeded.
    final stream = invoker(method, requests, options);
    return stream
      ..handleError(_handleError);
  }
}
```

**Constructor dependencies** — same as `AuthInterceptor`:
- `FlutterSecureStorage` for async token reads.
- `LogoutNotifier` for triggering session expiry (the `LogoutNotifier → UserNotifier.clearSession` chain fires exactly as in the Dio case).

### Stub Wiring

Generated stubs (produced by `protoc` with `dart-grpc` plugin) accept an `interceptors` list:

```dart
final channel = ClientChannel(
  host,
  port: port,
  options: const ChannelOptions(
    credentials: ChannelCredentials.insecure(), // or secure
  ),
);

final authInterceptor = GrpcAuthInterceptor(
  storage: const FlutterSecureStorage(),
  logoutNotifier: App.shared.logoutNotifier,
);

final mindClient = MindServiceClient(
  channel,
  interceptors: [authInterceptor],
);
```

Multiple interceptors are applied in list order (first to last wraps outermost to innermost). The interceptor is reused across all stub instances that share the same channel — pass the same `GrpcAuthInterceptor` instance to every stub rather than constructing one per stub.

In the existing DI pattern (`App.shared` / `App.dart`), `GrpcAuthInterceptor` should be instantiated once in `App._init()` alongside `AuthInterceptor`, and stored as a field. Each generated stub client is then instantiated with it.

## Open Questions

- The `interceptStreaming` async limitation: if gRPC streaming endpoints are added that require auth, a token cache field (updated on each successful unary call or on `UserNotifier` login events) will be needed. The cache introduces staleness risk but is the standard workaround for the synchronous `interceptStreaming` signature.
- ~~Whether NestJS `@nestjs/microservices` gRPC guard returns `UNAUTHENTICATED` (16) or `PERMISSION_DENIED` (7) for expired JWT~~ — **Resolved:** `@nestjs/microservices` maps `UnauthorizedException` → gRPC 16 `UNAUTHENTICATED` via its built-in `RpcExceptionFilter`. The current `JwtAuthGuard` throws only `UnauthorizedException`, so `StatusCode.unauthenticated` is the correct code to intercept. Note: the HTTP guard calls `context.switchToHttp()` and cannot be reused on gRPC controllers as-is — the dedicated `GrpcAuthInterceptor` (roadmap task 1.4) reads from gRPC metadata and throws the right exception type.
