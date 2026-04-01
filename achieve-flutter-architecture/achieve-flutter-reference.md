# Achieve Flutter Architecture — Code Reference

Complete code examples for each pattern. Copy and adapt for your features.

## 1. Repository Mixin (Caching)

```dart
typedef Operation<T> = Future<T> Function();
typedef SerializableOperation = Future<Map<String, dynamic>> Function();
typedef Deserializer<T> = T Function(Map<String, dynamic> data);

mixin RepositoryMixin {
  final Map<String, dynamic> _memoryCache = {};

  SharedPref get sharedPref => AppCore.instance.sharedPref;

  /// Persistent cache — survives app restarts (SharedPreferences)
  @protected
  Future<T> runPersistedQuery<T>(
    String key,
    bool remoteOnly,
    SerializableOperation operation,
    Deserializer<T> deserializer,
  ) async {
    return await _runSerializedQuery<T>(key, remoteOnly, operation, deserializer, false);
  }

  /// Secure cache — encrypted storage for sensitive data
  @protected
  Future<T> runSecureQuery<T>(
    String key,
    bool remoteOnly,
    SerializableOperation operation,
    Deserializer<T> deserializer,
  ) async {
    return await _runSerializedQuery<T>(key, remoteOnly, operation, deserializer, true);
  }

  /// Ephemeral cache — in-memory only, lost on app restart
  @protected
  Future<T> runEphemeralQuery<T>(String key, Operation<T> operation,
      {bool refresh = false}) async {
    if (refresh || !_memoryCache.containsKey(key)) {
      _memoryCache[key] = await runOperation(operation);
    }
    return _memoryCache[key] as T;
  }

  /// No cache — direct operation with error mapping
  @protected
  Future<T> runOperation<T>(Operation<T> operation) async {
    try {
      return await operation();
    } on NetworkError catch (e) {
      throw ServerOperationFailure(
        message: e.message,
        wrapped: e.toJson(),
        response: e.rawResponse,
      );
    } on OperationFailure catch (_) {
      rethrow;
    } catch (e, st) {
      AppCore.instance.logger.error("e: $e", trace: st);
      throw UnknownOperationFailure(e);
    }
  }

  Future<T> _runSerializedQuery<T>(
    String key, bool remoteOnly,
    SerializableOperation operation, Deserializer<T> deserializer, bool secure,
  ) async {
    try {
      if (!remoteOnly) {
        try {
          final cache = await sharedPref.getString(key, secure: secure);
          if (cache != null) {
            final data = json.decode(cache);
            return deserializer(data);
          }
        } catch (e) {
          AppCore.instance.logger.warning("Cache deserialization failed for $key: $e");
        }
      }
      final data = await runOperation(operation);
      await sharedPref.putString(key, json.encode(data), secure: secure);
      return deserializer(data!);
    } on OperationFailure catch (_) {
      rethrow;
    } catch (e, st) {
      AppCore.instance.logger.error("e: $e", trace: st);
      throw UnknownOperationFailure(e);
    }
  }

  @protected
  Future<void> clear() async {
    sharedPref.clear(clearSecure: true);
    _memoryCache.clear();
  }
}
```

## 2. Repository Example (Interface + Implementation)

```dart
// Abstract interface — defines the contract
abstract class SavingsAccountRepository {
  Future<List<SavingsAccount>> fetchAccounts(bool remoteOnly);
  Future<SavingsAccountDetail> fetchAccount(String id, bool remoteOnly);
  Future<SavingsAccount> createAccount(String name, String subscriptionId);
  Future<SavingsAccount> editAccount(String id, String name, {String? customColor});
}

// Implementation — mixes in RepositoryMixin for caching
class SavingsAccountRepositoryImpl
    with RepositoryMixin
    implements SavingsAccountRepository {

  ApiClient get _apiClient => AppCore.instance.apiClient;

  @override
  Future<List<SavingsAccount>> fetchAccounts(bool remoteOnly) async {
    return await runPersistedQuery("savings_accounts", remoteOnly, () async {
      final resultMap = await _apiClient.runQuery(
        FetchAccountsRequest(),
        resultKey: 'me',
      );
      return resultMap!;
    }, (data) {
      final list = data['savingsAccounts'] as Iterable;
      return list.map((e) => SavingsAccount.fromJson(e)).toList();
    });
  }

  @override
  Future<SavingsAccountDetail> fetchAccount(String id, bool remoteOnly) async {
    return await runPersistedQuery("savings_accounts:$id", remoteOnly, () async {
      final resultMap = await _apiClient.runQuery(
        FetchAccountRequest(id),
        resultKey: 'me',
      );
      return resultMap!['savingsAccounts'][0];
    }, (data) {
      return SavingsAccountDetail.fromJson(data);
    });
  }

  @override
  Future<SavingsAccount> createAccount(String name, String subscriptionId) async {
    // Mutations use runOperation (no cache)
    return await runOperation(() async {
      final resultMap = await _apiClient.runMutation(
        CreateAccountRequest(name, subscriptionId),
        resultKey: 'createAccount',
      );
      return SavingsAccount.fromJson(resultMap!['account']);
    });
  }

  @override
  Future<SavingsAccount> editAccount(String id, String name, {String? customColor}) async {
    return await runOperation(() async {
      final resultMap = await _apiClient.runMutation(
        EditAccountRequest(id, name: name, customColor: customColor),
        resultKey: 'editAccount',
      );
      return SavingsAccount.fromJson(resultMap!['account']);
    });
  }
}
```

## 3. DI Container Setup

```dart
final getIt = GetIt.instance;

Future<void> initCore() async {
  // App-level managers (lazy singletons — created on first access)
  getIt.registerLazySingleton<AppStateManager>(() => AppStateManager());
  getIt.registerLazySingleton<DeeplinkManager>(() => DeeplinkManager());

  // Core services (singletons — created immediately)
  getIt.registerSingleton<EventBus>(EventBus());
  getIt.registerSingleton<Scheduler>(SchedulerImpl());
  getIt.registerSingleton<AuthRepository>(
    AuthRepositoryImpl(uploaderRepository: getIt.get()),
  );

  _initFeatures();
}

void _initFeatures() {
  // LazySingleton — most repositories (shared state, created on first use)
  getIt.registerLazySingleton<GoalsRepository>(() => GoalsRepositoryImpl(getIt()));
  getIt.registerLazySingleton<WalletsRepository>(() => WalletRepositoryImpl());
  getIt.registerLazySingleton<SavingsAccountRepository>(() => SavingsAccountRepositoryImpl());

  // Factory — new instance each time (no shared state)
  getIt.registerFactory<LoansRepository>(() => LoansRepositoryImpl());
  getIt.registerFactory<NotificationsRepository>(() => NotificationsRepositoryImpl());
}
```

## 4. Event Bus

```dart
abstract mixin class EventBusListener {
  void onEvent(dynamic event);
}

class EventBus {
  Map<Type, Set<EventBusListener>> listeners = {};

  void registerListener<T>(EventBusListener listener) {
    if (!listeners.containsKey(T)) listeners[T] = {};
    listeners[T]!.add(listener);
  }

  void unregisterListener<T>(EventBusListener listener) {
    listeners[T]?.remove(listener);
  }

  void emit<T>(T event) {
    AppCore.instance.logger.debug('Event emitted: $event');
    if (!listeners.containsKey(T)) return;

    for (final listener in listeners[T]!) {
      try {
        listener.onEvent(event);
      } catch (e, st) {
        AppCore.instance.logger.error(
          "Event bus listener $listener failed for $event: $e", trace: st,
        );
      }
    }
  }
}
```

## 5. DataPage Base Class

DataPage wraps content in `FocusDetector`. FocusDetector fires `onFocusGained` whenever the page becomes visible — on first build, navigated back to via pop, app resumes from background, or tab switch. **Every DataPage automatically re-fetches data when the user returns to it. No manual reload is needed after push/pop navigation.**

Subclasses never call `setState` or override `initState`. Just assign fields in `onLoad()` and return widgets in `buildPage()`. DataPage handles all state internally.

**Two-layer loading design:**
- **Inline content during initial load:** Plain white `Container` — invisible against white backgrounds, no visual flash between pages.
- **Actual loading indicator:** `OverlayManager.pageBusy()` shows a global overlay spinner *on top of* the app (managed via EventBus). The page content stays blank underneath.
- **Error state:** Full-screen `DataError` widget with retry button (also on white background).
- This separation means the loading spinner is independent of page layout — it floats above, doesn't cause content shifts.

```dart
abstract class DataPage<T extends StatefulWidget>
    extends OperationRunnerState<T> with AutomaticKeepAliveClientMixin {

  bool loaded = false;
  bool loading = true;
  Object? lastError;

  bool get showLoader => true;
  bool get showError => !loading && !loaded && lastError != null;
  bool get isLoadingInitialData => loading && !loaded;

  /// Fetch data and assign to fields — called on first focus
  Future<void> onLoad();

  /// Re-fetch on subsequent visits — usually just calls onLoad()
  Future<void> onRefresh();

  /// Build the UI — only called when data is loaded
  Widget buildPage(BuildContext context);

  void refreshChildren() {
    getIt.get<EventBus>().emit(PageReloaded());
  }

  void requestRender() {
    if (!mounted) return;
    setState(() {});
  }

  Future<void> _loadData() async {
    loading = true;
    requestRender();

    try {
      if (!loaded) {
        if (showLoader) OverlayManager.pageBusy();
        await onLoad();
        loaded = true;
      } else {
        refreshChildren();
        await onRefresh();
      }
      lastError = null;
    } catch (e, st) {
      lastError = e;
      AppCore.instance.logger.error("Data load error: $e\n$st");
    }

    OverlayManager.pageIdle();
    loading = false;
    requestRender();
  }

  @override
  Widget build(BuildContext context) {
    super.build(context);
    return FocusDetector(
      child: showError
          ? buildError()
          : isLoadingInitialData
              ? Container(color: Colors.white)
              : buildPage(context),
      onFocusGained: () => _loadData(),
      onFocusLost: () {},
    );
  }

  Widget buildError() => Material(
    color: Colors.white,
    child: Center(child: DataError(onRetry: _loadData)),
  );

  @override
  bool get wantKeepAlive => false;
}
```

**Composition:** DataPage manages page-level lifecycle. For screens needing independent loading states per section, compose sub-widgets within `buildPage()` that manage their own data. DataPage does not force a single loading state on the whole screen.

**Typical usage — no initState, no setState, no manual reload:**

```dart
class _HomePageState extends DataPage<HomePage> {
  List<Program> _programs = [];

  @override
  Future<void> onLoad() async {
    _programs = await getIt.get<ProgramRepository>().getPrograms(studentId);
  }

  @override
  Future<void> onRefresh() => onLoad();

  @override
  Widget buildPage(BuildContext context) {
    return Scaffold(
      body: ListView.builder(
        itemCount: _programs.length,
        itemBuilder: (context, index) => Text(_programs[index].name),
      ),
    );
  }
}
```

## 6. OperationRunnerState

DataPage extends this. Use `runOperation()` for user-triggered actions (claim, submit, delete) with automatic loading overlay and error dialogs.

```dart
typedef Operation<T> = Future<T> Function();
typedef OperationFailureHandler = Future<bool> Function(OperationFailure e);

abstract class OperationRunnerState<S extends StatefulWidget> extends State<S>
    with PageWithAnalyticsMixin {

  @protected
  Future<T?> runOperation<T>(
    String name,
    Operation<T> operation, {
    Map<String, dynamic>? props,
    bool showLoader = true,
    OperationFailureHandler? errorHandler,
  }) async {
    T? result;
    try {
      if (showLoader) OverlayManager.pageBusy();
      logOperationEvent(name, OperationState.started(props: props));
      result = await operation();
      logOperationEvent(name, OperationState.success(props: props));
    } catch (e) {
      if (showLoader) OverlayManager.pageIdle();
      var message = "$e";

      if (e is OperationFailure) {
        logOperationEvent(name, OperationState.failed(e.message, props: props));
        message = e.message;
        if (((await errorHandler?.call(e)) ?? false)) return null;
      }

      if (mounted) await showError(context, message);
    } finally {
      if (showLoader) OverlayManager.pageIdle();
    }
    return result;
  }
}
```

**Usage in a DataPage:**
```dart
// User taps "Claim Reward" button
onTap: () async {
  final result = await runOperation('claim_reward', () async {
    return await rewardsRepo.claimReward(reward.id);
  });
  if (result != null) {
    // Success — update UI
  }
}
```

**Custom error handler — suppress default dialog for business-specific errors:**
```dart
await runOperation('create_session', () async {
  await authRepository.createSession(email, password);
}, errorHandler: (error) async {
  if (!error.message.toLowerCase().contains('deactivated')) return false;

  // Handle deactivated accounts with a custom reactivation flow
  if (await maybeActivateAccount()) {
    await authRepository.createSession(email, password);
  }
  return true; // suppress default error dialog
});
```

## 7. Child Widgets and PageReloaded

DataPage handles page-level data. For composed screens (balance card + goals list + transactions + promo banner), child widgets load their own data and listen for `PageReloaded`. When DataPage refreshes, it emits `PageReloaded` — each child independently decides whether to reload. The page doesn't know what child widgets it contains.

```dart
class _BalanceCardState extends State<BalanceCard>
    implements EventBusListener {
  double balance = 0;

  @override
  void initState() {
    super.initState();
    getIt.get<EventBus>().registerListener<PageReloaded>(this);
    _loadBalance();
  }

  @override
  void onEvent(event) {
    if (event is PageReloaded) _loadBalance();
  }

  Future<void> _loadBalance() async {
    balance = await getIt.get<WalletRepository>().getBalance();
    if (mounted) setState(() {});
  }

  @override
  void dispose() {
    getIt.get<EventBus>().unregisterListener<PageReloaded>(this);
    super.dispose();
  }
}
```

Add, remove, or rearrange child widgets without touching the page's `onLoad`.

## 8. Navigation

### Dual Routers

Guest router (login, registration, onboarding) and user router (everything else). They don't know about each other. Auth state change swaps the entire router — clears the nav stack naturally.

```dart
// In app.dart
if (authStateManager.isAuthenticated) {
  return UserHost(routerConfig: userRouter);
}
return GuestHost(routerConfig: guestRouter);
```

### Named Routes Only

```dart
abstract class UserRoutes {
  static const home = 'home';
  static const profile = 'profile';
  static const goalDetails = 'goal_details';
  // ...
}

// pushNamed — user should be able to go back
context.pushNamed(UserRoutes.goalDetails, pathParameters: {'id': goalId});

// goNamed — top-level transition, no back
context.goNamed(UserRoutes.home);
```

**Never:** `context.push('/profiles/new')` — always use named routes.

### Path Parameters for IDs

IDs in the path make routes bookmarkable and deep-linkable:

```dart
GoRoute(
  path: 'goals/:id',
  builder: (context, state) => GoalDetailsPage(
    goalId: state.pathParameters['id']!,
  ),
),
```

### Extra for Objects

Pass the full object when drilling down from a list — instant navigation, no re-fetch. Fall back to fetching by ID when `extra` is null (deep links, app restarts):

```dart
GoRoute(
  path: 'detail',
  builder: (context, state) =>
      PayslipDetailPage(payslip: state.extra as PayrollPaySlip),
),

// In the page — graceful fallback
class _PayslipDetailPageState extends DataPage<PayslipDetailPage> {
  late PayrollPaySlip payslip;

  @override
  Future<void> onLoad() async {
    payslip = widget.payslip ??
        await getIt.get<PayrollRepository>().getPayslip(widget.payslipId);
  }
}
```

### What Doesn't Go in Routes

Routes = navigation identity (which screen, which entity). Business state (current step in a flow, active filters) lives in a state manager or repository via GetIt.

## 9. Error Handling & Reporting

### End-to-End Error Flow

```
API call fails
  → NetworkErrorInterceptor maps DioError → NetworkError (user-friendly message)
  → GraphqlErrorInterceptor checks for UNAUTHENTICATED → pause, refresh token, retry
  → NetworkEventListener checks for 503 → emit maintenance event
  → RepositoryMixin.runOperation() wraps NetworkError → ServerOperationFailure
  → DataPage._loadData() catches → sets lastError, shows DataError with retry
  → OperationRunnerState.runOperation() catches → logs analytics, shows error dialog
  → AchieveLogger.error() → sends to PostHog, Sentry, Crashlytics
```

### Auth Error → Token Refresh → Retry (Pattern)

Auth detection varies by backend (error codes, HTTP status, etc.), but the pattern is consistent:

```dart
// In NetworkEventListener — handles auth errors with request queuing
Future<bool> onAuthError() async {
  // 1. Pause ALL in-flight requests (Dio request/response locks)
  _apiClient.pauseRequests();

  // 2. Attempt token refresh (uses separate Dio instance to bypass locks)
  final isSuccess = await _refreshSession();

  // 3. Resume — all queued requests retry automatically
  _apiClient.resumeRequests();

  if (isSuccess) return true;  // Retry original request

  // 4. Refresh failed → logout
  getIt.get<EventBus>().emit(NetworkStateChanged.authTokenExpired());
  return false;
}
```

### App State Transitions from Errors

```dart
// In AppStateManager — listens for network state changes
Future<void> onEvent(event) async {
  if (event is NetworkStateChanged) {
    switch (event.state) {
      case NetworkState.maintenance:
        _appState = AppState.maintenance;  // Shows MaintenanceHost screen
        getIt.get<EventBus>().emit(AppStateChanged.changed());

      case NetworkState.authTokenExpired:
        if (appState == AppState.authenticated) logout();  // Full cleanup + logout
    }
  }
}
```

### Logger — Remote vs Local

```dart
// Only error level sends to remote services
switch (logLevel) {
  case LogLevel.error:
    _crashlytics?.recordError(data, trace);
    _posthog?.captureException(error: data, stackTrace: trace);
    Sentry.captureException(data, stackTrace: trace);
    break;
  default:
    // debug, info, warning — console only
}
```

### NetworkError — Context-Aware Messages

```dart
class NetworkError {
  String message;        // User-facing ("A network error occurred. Please check your connection.")
  String? fullError;     // Validation errors joined for display
  dynamic rawResponse;   // Full server response for logging

  factory NetworkError.fromDio(DioError dioErr) {
    switch (dioErr.type) {
      case DioErrorType.connectTimeout:
      case DioErrorType.sendTimeout:
        message = 'Request timed out';
      case DioErrorType.receiveTimeout:
        message = 'Response timed out';
      case DioErrorType.other:
        if (message.contains('SocketException') || message.contains('failed host lookup')) {
          message = 'A network error occurred. Please check your connection.';
        }
      // ...
    }
  }
}
```

### Operation Analytics Tracking

```dart
// Every runOperation call tracks a 3-state lifecycle
logOperationEvent(name, OperationState.started(props: props));   // initiated
logOperationEvent(name, OperationState.success(props: props));   // success
logOperationEvent(name, OperationState.failed(e.message, props: props));  // failed + reason

// Events broadcast to ALL analytics: Firebase, Amplitude, PostHog, AppsFlyer, Facebook
```

### Custom Error Handler Callback

```dart
// Suppress the default error dialog for specific failures
await runOperation('withdraw', () async {
  return await walletRepo.withdraw(amount);
}, errorHandler: (failure) async {
  if (failure.message.contains('insufficient funds')) {
    // Handle specifically — show custom UI instead of generic error dialog
    showInsufficientFundsSheet(context);
    return true;  // true = suppress default showError dialog
  }
  return false;  // false = show default error dialog
});
```

### OverlayManager — Event-Driven Loading

OverlayManager wraps the entire app in a `Stack`. It listens for `PageBusyStateChanged` events via EventBus and shows/hides a `LoaderView` overlay on top. While busy, the back button is blocked.

Both `DataPage._loadData()` and `OperationRunnerState.runOperation()` call `pageBusy`/`pageIdle` — the overlay is shared and global.

```dart
class OverlayManager extends StatefulWidget {
  const OverlayManager({super.key, required this.child});
  final Widget child;

  // Static methods emit events — any code can trigger the overlay
  static void pageBusy() {
    getIt.get<EventBus>().emit(PageBusyStateChanged(true));
  }
  static void pageIdle() {
    getIt.get<EventBus>().emit(PageBusyStateChanged(false));
  }

  @override
  State<OverlayManager> createState() => _OverlayManagerState();
}

class _OverlayManagerState extends State<OverlayManager>
    with SingleTickerProviderStateMixin
    implements EventBusListener {
  bool busy = false;
  EventBus eventBus = getIt<EventBus>();
  late AnimationController loaderAnimationController;

  @override
  void initState() {
    super.initState();
    loaderAnimationController = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 120),
    );
    eventBus.registerListener<PageBusyStateChanged>(this);
  }

  @override
  Widget build(BuildContext context) {
    return WillPopScope(
      onWillPop: () async => !busy,  // Block back button while loading
      child: Stack(
        children: [
          widget.child,               // App content underneath
          if (busy)
            LoaderView(controller: loaderAnimationController),  // Overlay on top
        ],
      ),
    );
  }

  @override
  onEvent(event) {
    if (event is PageBusyStateChanged) {
      busy = event.busy;
      setState(() {});
    }
  }

  @override
  void dispose() {
    eventBus.unregisterListener<PageBusyStateChanged>(this);
    loaderAnimationController.dispose();
    super.dispose();
  }
}
```

**Setup:** OverlayManager must wrap the app's root widget (typically in `app.dart` or `MaterialApp` builder) so the overlay covers all pages.

## 10. Network Client Interface

```dart
abstract class ApiClient {
  /// Read operations
  Future<Map<String, dynamic>?> runQuery(ApiRequest request, {String? resultKey});

  /// Write operations
  Future<Map<String, dynamic>?> runMutation(ApiRequest request, {String? resultKey});

  /// Pause/resume for auth flows
  void pauseRequests();
  void resumeRequests();
  void cancelRequests();

  /// Token refresh using separate client (bypasses interceptor lock)
  Future<Map<String, dynamic>?> refreshToken(ApiRequest request, {String? resultKey});
}
```

Implementation adds interceptors: `AuthInterceptor` (bearer token), `ErrorInterceptor` (maps API errors), `NetworkErrorInterceptor` (maps connectivity issues), `RequestLoggingInterceptor` (debug logging).

## 11. Error Hierarchy

```dart
abstract class Failure extends Equatable {
  const Failure([this.properties = const []]);
  final List properties;

  @override
  List<Object> get props => [properties];
}

class OperationFailure extends Failure {
  OperationFailure({this.message = 'Operation Failure', this.wrapped, this.response})
      : super([message, wrapped, response]);
  final dynamic message;
  final Object? wrapped;
  final Object? response;

  @override
  String toString() => "${wrapped != null ? 'wrapped($wrapped)->' : ''}msg($message)";
}

class ServerOperationFailure extends OperationFailure {
  ServerOperationFailure({super.message = 'Server Failure', super.wrapped, super.response});
}

class UnknownOperationFailure extends OperationFailure {
  UnknownOperationFailure(wrapped) : super(message: 'Something went wrong', wrapped: wrapped);
}

class CacheOperationFailure extends OperationFailure {
  CacheOperationFailure({super.message = 'Cache Failure', super.wrapped});
}
```

## 12. Model Example

```dart
import 'package:equatable/equatable.dart';
import 'package:json_annotation/json_annotation.dart';
import 'package:meta/meta.dart';

part 'reward.g.dart';

enum RewardStatus { available, claimed, expired, unknown }

@immutable
@JsonSerializable()
class Reward extends Equatable {
  const Reward(this.id, this.title, this.description, this.points, this.status, this.createdAt);

  final String id;
  final String title;
  final String? description;
  final int points;

  @JsonKey(unknownEnumValue: RewardStatus.unknown)
  final RewardStatus status;

  final DateTime createdAt;

  factory Reward.fromJson(Map<String, dynamic> json) => _$RewardFromJson(json);
  Map<String, dynamic> toJson() => _$RewardToJson(this);

  @override
  List<Object?> get props => [id, title, points, status];
}
```

## 13. Test Example

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mocktail/mocktail.dart';

class MockApiClient extends Mock implements ApiClient {}
class MockSharedPref extends Mock implements SharedPref {}
class MockLogger extends Mock implements AppLogger {}

void main() {
  late ApiClient apiClient;
  late SharedPref sharedPref;

  setUpAll(() {
    final mockLogger = MockLogger();
    AppCore.instance.registerMock<AppLogger>(mockLogger);
  });

  setUp(() {
    apiClient = MockApiClient();
    AppCore.instance.registerMock<ApiClient>(apiClient);

    sharedPref = MockSharedPref();
    AppCore.instance.registerMock<SharedPref>(sharedPref);

    registerFallbackValue(const FetchRewardsRequest());
  });

  test("fetchRewards returns parsed list", () async {
    final repo = RewardsRepositoryImpl();

    when(() => apiClient.runQuery(
      any(that: isA<FetchRewardsRequest>()),
      resultKey: 'me',
    )).thenAnswer(
      (_) => Future.value({
        'rewards': RewardFixture.factory().makeMany(3).map((e) => e.toJson()).toList(),
      }),
    );

    when(() => sharedPref.putString('rewards', any())).thenAnswer((_) => Future.value());

    final result = await repo.fetchRewards(true);

    expect(result, isA<List<Reward>>());
    expect(result.length, 3);
  });

  test("claimReward returns claimed reward", () async {
    final repo = RewardsRepositoryImpl();

    when(() => apiClient.runMutation(
      any(that: isA<ClaimRewardRequest>()),
      resultKey: 'claimReward',
    )).thenAnswer(
      (_) => Future.value({
        'reward': RewardFixture.factory().makeSingle().toJson(),
      }),
    );

    final result = await repo.claimReward('reward-1');
    expect(result, isA<Reward>());
  });
}
```

## 14. Project Setup Checklist

1. Create Flutter project: `flutter create --org com.example app_name`
2. Add dependencies: `get_it`, `equatable`, `json_annotation`, `json_serializable`, `build_runner`, `mocktail`, `data_fixture_dart`, `focus_detector`
3. Create folder structure per Project Structure in SKILL.md
4. Implement `RepositoryMixin` (copy from section 1)
5. Implement `EventBus` (copy from section 4)
6. Implement `DataPage` + `OperationRunnerState` (copy from sections 5-6)
7. Implement error hierarchy (copy from section 8)
8. Implement or integrate API client (section 7)
9. Create `injection_container.dart` with `initCore()` (section 3)
10. Wire up in `main.dart`: initialize core, run app
