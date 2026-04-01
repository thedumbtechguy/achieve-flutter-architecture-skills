---
name: achieve-flutter-architecture
description: Use when building Flutter apps following the Achieve architecture — repository pattern with caching mixins, service locator DI, event bus communication, and DataPage lifecycle for screens
---

# Achieve Flutter Architecture

## Overview

Every feature follows the same shape: **Model → Repository → Screen → Route.** A screen doesn't catch errors, manage loading spinners, or know auth tokens exist. It calls a repository, assigns the result to a field, and returns widgets. Everything else — loading states, error handling, caching, auth refresh, analytics — is the architecture's problem.

## When to Use

- Building a Flutter app that will grow beyond a few screens
- Adding a feature (screen, repository, model) following this pattern
- Setting up DI, routing, or network layers

**Don't use for:** simple prototypes, projects with established different architecture (BLoC, Riverpod), package/plugin development.

## Project Structure

```
features/<feature>/
  └── <feature>_repository.dart  # Abstract interface + implementation

core/models/
  └── <model>.dart               # Data model

ui/pages/user/<feature>/
  ├── <feature>_list_page.dart   # List screen (extends DataPage)
  ├── <feature>_detail_page.dart # Detail screen (extends DataPage)
  └── widgets/                   # Child widgets (may listen to PageReloaded)

services/
  ├── injection_container.dart   # DI setup (GetIt)
  ├── event_bus/                 # Cross-feature events
  └── core/                      # Network clients, logging, config

router/
  ├── user_router.dart           # Authenticated routes
  └── guest_router.dart          # Unauthenticated routes
```

## Data Layer

### Repository + Caching Mixin

Each feature: **abstract interface** + **implementation** mixing in `RepositoryMixin`. Four strategies:

| Strategy | Method | Use for |
|----------|--------|---------|
| Persistent | `runPersistedQuery` | Data that doesn't change every second (goals, profiles). Returns cached version immediately; pass `remoteOnly: true` for fresh fetch |
| Secure | `runSecureQuery` | Same pattern, encrypted storage. Auth tokens, balances, transactions |
| Ephemeral | `runEphemeralQuery` | In-memory only, session-scoped. Search results, filtered lists |
| None | `runOperation` | Mutations. You don't cache writes |

Cache is a performance optimization, not correctness. If cached data fails to deserialize (schema change after app update), it silently falls through to network fetch.

### GetIt — Service Locator

Repositories registered as lazy singletons against abstract interfaces. Accessed via `getIt.get<GoalRepository>()`. We chose GetIt over Riverpod/Provider because repositories need to be accessible **outside the widget tree** — in interceptors, in other repositories, in utility classes. GetIt doesn't care about `BuildContext`.

## Screen Layer

### DataPage — The Screen Lifecycle

**Every screen loading async data MUST extend DataPage.** No `initState`, no `setState`, no try/catch.

```dart
class _GoalListPageState extends DataPage<GoalListPage> {
  List<Goal> goals = [];

  @override
  Future<void> onLoad() async {
    goals = await getIt.get<GoalRepository>().fetchGoals();
  }

  @override
  Future<void> onRefresh() => onLoad();

  @override
  Widget buildPage(BuildContext context) {
    return ListView.builder(
      itemCount: goals.length,
      itemBuilder: (_, i) => GoalTile(goal: goals[i]),
    );
  }
}
```

DataPage provides: loading state until `onLoad` completes, error widget with retry if it throws, only calls `buildPage` when data is ready, manages `setState` internally.

`onLoad` is called on first focus. `onRefresh` on every subsequent focus — navigate back, tab switch, app resume. DataPage wraps content in `FocusDetector` which fires automatically. This eliminates "I created a goal but it didn't show up in the list" — every DataPage refreshes on focus.

**Two-layer loading:** The page shows a plain white `Container` during initial load (invisible against white backgrounds — no flash). The actual spinner is **OverlayManager** — a global overlay on top of the app. Why app-level? Because screens can dispose during async operations. User taps Submit then navigates back — per-screen overlay crashes or leaks. App-level OverlayManager outlives every screen.

### OperationRunnerState — User Actions

DataPage extends OperationRunnerState, so every screen gets both reads and writes:

```dart
await runOperation('redeem_reward', () async {
  await rewardRepository.redeemReward(id);
  await onRefresh();
});
```

`runOperation` handles: loading overlay, hiding it when done, catching errors, showing error dialog, logging analytics. The `errorHandler` callback suppresses the default dialog for business-specific errors (return `true` = handled, `false` = show default dialog).

### Child Widgets and PageReloaded

DataPage handles page-level data. For composed screens (balance card + goals list + transactions + promo banner), child widgets load their own data and listen for `PageReloaded`:

```dart
class _BalanceCardState extends State<BalanceCard> implements EventBusListener {
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
}
```

The page doesn't know what child widgets it contains. It emits "I refreshed" and each child independently reloads. Add, remove, or rearrange widgets without touching `onLoad`.

## Navigation

**Dual routers:** Guest router (login, registration) and user router (everything else). Auth state change swaps the entire router — clears the nav stack naturally. No redirect guards scattered across screens.

**Named routes only:** `context.pushNamed(UserRoutes.goalDetails, pathParameters: {'id': goalId})`. Never raw paths.
- `pushNamed` — user should be able to go back
- `goNamed` — top-level transition, no back

**`extra` for objects:** Pass the full object when drilling down from a list (instant navigation, no re-fetch). Fall back to fetching by ID when `extra` is null (deep links, app restarts):

```dart
@override
Future<void> onLoad() async {
  payslip = widget.payslip ??
      await getIt.get<PayrollRepository>().getPayslip(widget.payslipId);
}
```

**Routes are for navigation identity** (which screen, which entity), not transient state (current step, active filters). That lives in a state manager or repository via GetIt.

## Error Flow

No screen handles infrastructure errors. Three layers do it:

**Layer 1 — Network Interceptors:** Auth expiry → pause all requests, refresh token, retry. 503 → emit maintenance event, app transitions. Screens never see these.

**Layer 2 — Repository Mixin:** `runOperation` wraps raw exceptions into typed `OperationFailure` with human-readable messages. UI never sees stack traces.

**Layer 3 — OperationRunnerState:** Catches `OperationFailure`, hides overlay, logs analytics (operation name + failure reason), shows error dialog. Optional `errorHandler` callback for business-specific errors.

**What screens don't handle:** Auth errors, 503s, raw exceptions, error UI, loading state for operations. A screen provides the happy path and optionally one or two business-specific error handlers. Everything else is the architecture's problem.

## Building a New Feature

1. **Model** — `@JsonSerializable()` + `Equatable` in `core/models/`
2. **Repository** — abstract interface + implementation with `RepositoryMixin` in `features/<feature>/`
3. **Register** — one line in `injection_container.dart` (lazySingleton for most)
4. **Screen** — extend `DataPage`, implement `onLoad`/`onRefresh`/`buildPage`
5. **Route** — register in router

That screen gets loading states, error handling, caching, auto-refresh on focus, loading overlay during mutations, error dialogs, and analytics — without writing any of it.

## Testing Pattern

1. `setUpAll` — register mock logger
2. `setUp` — register mock network client + shared prefs via `registerMock<T>()`
3. Register fallback values for matchers
4. Create repository instance directly (not from DI)
5. Mock responses with `when().thenAnswer()`
6. Use `data_fixture_dart` for realistic test data

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Custom loading/error/refresh in pages | **Extend DataPage** — it handles all of this |
| Manual operation handling in pages | **Use runOperation()** — overlay, errors, analytics included |
| Catching errors in DataPage subclasses | Let DataPage handle it — shows DataError with retry |
| Per-screen loading overlay | Use **OverlayManager** — survives screen disposal |
| Skipping repository interface | Always define abstract class first |
| Wrong cache strategy | Persistent=stable, secure=sensitive, ephemeral=session, none=mutations |
| Fat pages with business logic | Keep logic in repository; pages call repos and render |
| Putting all child data in page onLoad | Use **PageReloaded** event — child widgets load their own data |
| Direct cross-feature calls | Use event bus for loose coupling |
| Using raw paths for navigation | Named routes with constants only |
| Business state in route params | Routes = navigation identity. Transient state lives in GetIt |

## If Starting a New Project

Same architecture, modern Dart 3 upgrades:

- **Freezed** for models — one annotation replaces JsonSerializable + Equatable + manual copyWith
- **Sealed classes + Result type** for errors — compiler enforces exhaustive handling instead of runtime hope
- **go_router_builder** for navigation — generated type-safe route methods, compile-time parameter validation

These are implementation upgrades, not architectural changes. The five pieces — repository with caching mixin, DataPage, OperationRunnerState with OverlayManager, EventBus, GetIt — stay the same.

See achieve-flutter-reference.md for complete code examples.
