---
type: pattern
severity: recommended
status: stable
applies_to: any-flutter-project
keywords: [dio, api-client, http, interceptors, config, environment, base-url, log-interceptor, dart-define]
related: [[layered-architecture]], [[adr-005-secure-storage-for-tokens]], [[async-init-provider-override]]
code_refs:
  - go_sport/lib/core/network/api_client.dart
  - go_sport/lib/core/config/app_config.dart
  - go_sport/lib/core/network/interceptors/auth_interceptor.dart
---

# API client with config and interceptors

## Problem

An app needs a single HTTP client used by every data-layer repository, configured per environment (dev/prod base URL, timeouts, log verbosity) and threading concerns through cross-cutting interceptors (auth header injection, refresh-on-401, retry, logging). Constructing Dio at each call site or scattering `BaseOptions` across files leads to drift; injecting raw Dio everywhere makes interceptor changes touch many files.

## Solution

A single `ApiClient` class wraps `Dio`. It takes two constructor arguments — an `AppConfig` (environment-derived options) and a `List<Interceptor>` (cross-cutting concerns) — and builds its Dio instance once. Repositories use `ApiClient`'s thin `get/post/put/delete` surface, never seeing Dio directly.

```dart
// core/config/app_config.dart
enum AppEnv { dev, prod }

class AppConfig {
  final AppEnv env;
  final String apiBaseUrl;
  final Duration connectTimeout;
  final Duration receiveTimeout;
  final bool enableLogging;

  const AppConfig._({ /* ... */ });

  static const dev = AppConfig._(
    env: AppEnv.dev,
    apiBaseUrl: 'http://...',
    connectTimeout: Duration(seconds: 30),
    receiveTimeout: Duration(seconds: 30),
    enableLogging: true,
  );

  static const prod = AppConfig._(
    env: AppEnv.prod,
    apiBaseUrl: 'https://api.gosport.com/v1',
    connectTimeout: Duration(seconds: 15),
    receiveTimeout: Duration(seconds: 15),
    enableLogging: false,
  );
}

// core/network/api_client.dart
class ApiClient {
  final Dio _dio;

  ApiClient(AppConfig config, List<Interceptor> interceptors)
      : _dio = Dio(BaseOptions(
          baseUrl: config.apiBaseUrl,
          connectTimeout: config.connectTimeout,
          receiveTimeout: config.receiveTimeout,
        )) {
    _dio.interceptors.addAll(interceptors);
    if (config.enableLogging) {
      _dio.interceptors.add(LogInterceptor(responseBody: true, requestBody: true));
    }
  }

  Future<Response> get(String path, { /* ... */ }) => _dio.get(/* ... */);
  // post, put, delete
}
```

Environment selection happens once in `main()` via `--dart-define=ENV=prod`:

```dart
const env = String.fromEnvironment('ENV', defaultValue: 'dev');
final config = env == 'prod' ? AppConfig.prod : AppConfig.dev;

final tokenStorage = TokenStorage();
await tokenStorage.init();

final apiClient = ApiClient(config, [AuthInterceptor(tokenStorage)]);
```

`ApiClient` is then exposed as a Riverpod provider with override (see [[async-init-provider-override]]). Repository implementations consume it via `ref.watch(apiClientProvider)`.

Interceptors get their dependencies through their own constructors — e.g., `AuthInterceptor(tokenStorage)` — and apply per-request behavior. A minimal `AuthInterceptor` attaches the Bearer header unless the request is marked public:

```dart
class AuthInterceptor extends Interceptor {
  final TokenStorage _tokenStorage;
  AuthInterceptor(this._tokenStorage);

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    if (options.extra['public'] != true) {
      final accessToken = _tokenStorage.accessToken;
      if (accessToken != null) {
        options.headers['Authorization'] = 'Bearer $accessToken';
      }
    }
    handler.next(options);
  }
}
```

## When to apply

- Any project with more than one repository talking to the same backend
- When dev/prod (or staging/canary/etc.) need different endpoints or timeout profiles
- When at least one cross-cutting HTTP concern (auth header, logging, retry) exists

## When NOT to apply

- One-off scripts or prototypes with a single endpoint — `Dio()` inline is fine
- Apps with multiple unrelated backends — one `ApiClient` per backend, not one giant wrapper

## Trade-offs

- (+) Single place to change baseURL, timeouts, or interceptor stack
- (+) Repositories test cleanly by overriding `apiClientProvider` with a fake `ApiClient`
- (+) `enableLogging` flag lets `LogInterceptor` be conditional (verbose in dev, silent in prod) without `if (kDebugMode)` scattering
- (+) `--dart-define=ENV=prod` keeps env selection at build time; runtime branches never see the wrong config
- (-) Indirection — a developer chasing "where does this header come from" has to find the interceptor, not grep for the header string
- (-) The thin `get/post/put/delete` surface is just enough for current needs; features that want Dio-specific options (streamed responses, cancellation tokens) either grow the wrapper or expose Dio
- (-) Construction order matters — interceptors that depend on async-initialized objects (`AuthInterceptor` needs an initialized `TokenStorage`) require `main()` to wait on the dependency first

## Common pitfalls

- Constructing `ApiClient` inside a `Provider((ref) => ...)` factory instead of `main()` — defeats the point if any interceptor needs an async-initialized dependency
- Adding `LogInterceptor` unconditionally — verbose logs in production leak tokens and slow request handling
- Letting features import Dio types directly (`Response`, `DioException`) — re-couples features to the HTTP library; map to app-level errors in `data/` and expose only domain types
- Forgetting `extra['public'] = true` on truly public endpoints (login, register, password reset) — `AuthInterceptor` attaches Bearer headers even there, which some servers reject
