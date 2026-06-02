# Plan: Phase 3 — Security Layer

**Created:** 2026-06-02  
**Status:** Approved — Ready for Implementation (pending Phase 1 and Phase 2 completion)  
**Parent Plan:** `Documentation/Plans/Plan_Initial_App_Delivery.md`  
**Prerequisite:** Phase 1 and Phase 2 complete and approved by df-qa.  
**Active Phase:** Phase 3 — Security Layer

---

## Objective

Implement phone credential authentication (biometric or device PIN) on every cold launch, and an idle timeout that returns the user to a lock screen after a configurable period of inactivity. No health data is accessible without a successful authentication. This phase delivers a hardened security layer on top of the encrypted storage foundation from Phase 2.

---

## Constraints

- Authentication uses device credentials only (`local_auth` package — biometric or PIN/pattern). Custom PIN management is out of scope.
- Lock screen is shown on cold launch, on idle timeout, and when the app returns from background after the idle threshold has elapsed.
- The idle timeout clock **pauses** while the lock screen is visible and **resets** on successful authentication (confirmed in parent plan Assumption 11).
- Idle timeout duration is configurable by the user via the Settings screen. Default: 5 minutes.
- Idle timeout setting is stored **unencrypted** in shared preferences (no health data; confirmed in parent plan Assumption 8).
- If the device has no enrolled credentials, show a clear user-facing message and do not lock the app (graceful degradation per parent plan deliverables).
- Android-first; iOS not in scope for v1.
- Rapid delivery and low complexity are explicit priorities (CODING_STANDARDS).

---

## Assumptions

1. Phase 1 navigation shell and Phase 2 encrypted storage are complete before implementation starts.
2. `local_auth ^2.3.0` (or latest compatible with minSdk 23) is used for biometric/device PIN.
3. `shared_preferences ^2.3.0` (or latest compatible) is used for the idle timeout setting (unencrypted).
4. The lock screen route is `/lock`, already defined as a named route in Phase 1.
5. The `LockScreen` widget created as a placeholder in Phase 1 is replaced with a functional implementation here.
6. "Cold launch" means the app process is starting fresh. Returning from background after idle threshold also triggers the lock screen.
7. The idle timer is driven by user interaction events (touch/tap). No touch for `N` minutes triggers the lock. A `GestureDetector` or `Listener` at the root `MaterialApp` level is the recommended approach.
8. The idle timer is managed by `IdleTimeoutService`, which is a singleton accessible via Riverpod provider.
9. Idle timeout duration is stored under key `'idle_timeout_minutes'` in shared preferences.
10. `NotifierProvider<IdleTimeoutNotifier, IdleTimeoutState>` is used for `IdleTimeoutService` (synchronous state — per CODING_STANDARDS). `SettingsService` uses `AsyncNotifierProvider` since SharedPreferences loading is async.
11. Lock screen does not provide a "cancel" or "back out" option. Authentication is mandatory.
12. Background detection uses `AppLifecycleListener` (Flutter 3.13+) or `WidgetsBindingObserver` to catch `AppLifecycleState.paused` / `resumed` transitions.

---

## Open Questions

| ID  | Question                                                                                     | Status |
| --- | -------------------------------------------------------------------------------------------- | ------ |
| OQ1 | Should background-to-foreground always trigger re-authentication, or only if idle threshold has elapsed while in background? | **Resolved** — Threshold-based: if background time ≥ idle threshold → lock on foreground return. If background < idle threshold → resume timer with remaining time. Stage 6 implements this. |
| OQ2 | What message should be shown when the device has no enrolled credentials? Exact copy TBD.   | **Resolved** — Placeholder text acceptable for Phase 3. Non-blocking. |

---

## Package Dependencies

Additions to `pubspec.yaml`:

```yaml
dependencies:
  local_auth: ^2.3.0
  shared_preferences: ^2.3.0
```

---

## Directory Structure (additions to Phase 2 scaffold)

```
lib/
  core/
    auth/
      auth_service.dart             # local_auth wrapper (abstract interface)
      auth_service_local.dart       # LocalAuthentication implementation
    idle/
      idle_timeout_service.dart     # IdleTimeoutService + IdleTimeoutState
      idle_timeout_provider.dart    # NotifierProvider
  features/
    auth/
      screens/
        lock_screen.dart            # Functional replacement of Phase 1 placeholder
      providers/
        auth_provider.dart          # AsyncNotifierProvider for auth state
    settings/
      screens/
        settings_screen.dart        # Add idle timeout duration option (replaces placeholder)
      providers/
        settings_provider.dart      # AsyncNotifierProvider for settings state
      services/
        settings_service.dart       # SharedPreferences read/write for settings
test/
  core/
    auth/
      auth_service_test.dart
    idle/
      idle_timeout_service_test.dart
  features/
    settings/
      settings_service_test.dart
```

---

## Stage Plan

### Stage 1 — Add Dependencies

Add `local_auth` and `shared_preferences` to `pubspec.yaml`. Run `flutter pub get`.

**Android setup required:**
- Add `<uses-permission android:name="android.permission.USE_BIOMETRIC"/>` and `<uses-permission android:name="android.permission.USE_FINGERPRINT"/>` to `AndroidManifest.xml` (check `local_auth` docs for the exact set required at pinned version).
- Verify `FlutterFragmentActivity` is the activity class in `AndroidManifest.xml` (required by `local_auth`). If `FlutterActivity` is present, replace it.

**Owner:** df-coder  
**Completion Criteria:** `pubspec.yaml` updated; `flutter pub get` succeeds; `AndroidManifest.xml` updated; main activity is `FlutterFragmentActivity`.

---

### Stage 2 — AuthService

Define abstract interface `AuthService` in `lib/core/auth/auth_service.dart`:

```dart
abstract class AuthService {
  Future<bool> authenticate({required String reason});
  Future<bool> isDeviceSupported();
}
```

`isDeviceSupported()` is the credential-availability guard: returns `false` if no secure lock screen (PIN, pattern, or biometric) is enrolled. `canCheckBiometrics` is **not** included — it checks only biometric hardware, not whether any credentials are enrolled, and is out of scope for Phase 3.

Implement `AuthServiceLocal` in `auth_service_local.dart` using `local_auth`'s `LocalAuthentication`.

- `authenticate` calls `localAuth.authenticate(localizedReason: reason, options: const AuthenticationOptions(biometricOnly: false))`. `biometricOnly: false` allows PIN/pattern fallback.
- `isDeviceSupported` delegates to `localAuth.isDeviceSupported()`.

Add `AsyncNotifierProvider<AuthNotifier, AuthState>` in `lib/features/auth/providers/auth_provider.dart`. `AuthState` has values: `locked`, `authenticated`, `unavailable`.

**Owner:** df-coder  
**Completion Criteria:** Interface, implementation, and provider stub compile cleanly.

---

### Stage 3 — Lock Screen UI

Replace the Phase 1 `LockScreen` placeholder with a functional implementation:

- Centred layout: app name/logo, "Unlock" button.
- On "Unlock" tap: call `AuthService.authenticate(reason: 'Unlock Symptom Diary')`.
- On success: update `AuthNotifier` state to `authenticated`, navigate to `/` (home).
- On failure: show a `SnackBar` with a retry message; stay on lock screen.
- If `isDeviceSupported()` is false: replace "Unlock" button with a message explaining that no credentials are enrolled (OQ2 — use placeholder text for now).
- No back button / no dismiss gesture. Use `PopScope(canPop: false, onPopInvokedWithResult: (_, __) {})` to block back navigation on the lock screen. (`WillPopScope` is removed in Flutter 3.24+ and must not be used.)

**Owner:** df-coder  
**Completion Criteria:** Lock screen displays; authenticate button calls `local_auth`; success navigates away; failure stays; back navigation blocked.

---

### Stage 4 — Cold Launch Lock

Ensure the app always shows the lock screen on cold launch before any health data is visible:

- In `main.dart` or `app.dart`, the initial route is `/lock` unconditionally.
- After successful authentication, navigate to `/` (replacing the lock route so back does not return to it).
- `EncryptedStorage.init()` (Phase 2) is still called before `runApp`; encrypted boxes are open but UI is not shown until authentication passes.

**Owner:** df-coder  
**Completion Criteria:** App opens to lock screen on fresh launch. After authentication, navigates to home. Back from home does not return to lock screen.

---

### Stage 5 — IdleTimeoutService

Implement `IdleTimeoutService` in `lib/core/idle/idle_timeout_service.dart`:

- Holds a `Timer` that fires after the configured duration with no activity.
- `resetTimer()` — cancels and restarts the timer. Called on every user interaction.
- `pauseTimer()` — cancels the timer without restarting. Called when lock screen is shown.
- `resumeTimer()` — restarts the timer. Called after successful authentication.
- On timer fire: set `isLocked = true` in `IdleTimeoutState`. Cancel the timer. **Do not navigate directly** — `IdleTimeoutService` has no `BuildContext` or `NavigatorKey`; navigation is handled by the widget layer.

`IdleTimeoutState` (immutable): `{ bool isLocked, Duration timeout }`.

Expose via `NotifierProvider<IdleTimeoutNotifier, IdleTimeoutState>` in `idle_timeout_provider.dart` (synchronous state — per CODING_STANDARDS `NotifierProvider` preference).

Wrap the root `MaterialApp` widget body with a `Listener` (not `GestureDetector`, to avoid interfering with child tap events):

```dart
Listener(
  behavior: HitTestBehavior.translucent,
  onPointerDown: (_) => ref.read(idleTimeoutProvider.notifier).resetTimer(),
  child: MaterialApp(...),
)
```

In `app.dart` (`ConsumerStatefulWidget`), add a `GlobalKey<NavigatorState>` to the state class and pass it to `MaterialApp`. Use it in a `ref.listen` on `idleTimeoutProvider` to handle navigation when `isLocked` transitions to `true`:

```dart
// In _AppState:
final GlobalKey<NavigatorState> _navigatorKey = GlobalKey<NavigatorState>();

// In build():
ref.listen<IdleTimeoutState>(idleTimeoutProvider, (prev, next) {
  if (next.isLocked && !(prev?.isLocked ?? false)) {
    ref.read(authProvider.notifier).lock();
    _navigatorKey.currentState?.pushNamedAndRemoveUntil('/lock', (_) => false);
  }
});

// MaterialApp receives the key:
Listener(
  behavior: HitTestBehavior.translucent,
  onPointerDown: (_) => ref.read(idleTimeoutProvider.notifier).resetTimer(),
  child: MaterialApp(
    navigatorKey: _navigatorKey,
    ...
  ),
)
```

`_navigatorKey.currentState` is the `NavigatorState` created inside `MaterialApp`. Using a `GlobalKey<NavigatorState>` avoids calling `Navigator.of(context)` from a context that is above `MaterialApp` (which would throw `Navigator operation requested with a context that does not include a Navigator`).

**State relationship note:** `IdleTimeoutState.isLocked` is the timer-trigger signal. `AuthState` in `auth_provider.dart` is the persistent auth state. The `ref.listen` handler above observes `isLocked`, calls `authNotifier.lock()` to sync the auth state, then navigates. After successful authentication on the lock screen, call `idleTimeoutNotifier.resumeTimer()` which resets `isLocked` to `false`.

**Owner:** df-coder  
**Completion Criteria:** Timer resets on interaction. Lock triggers after `N` minutes of inactivity (via `isLocked → ref.listen → navigate`). `pauseTimer` and `resumeTimer` behave correctly around the lock screen.

---

### Stage 6 — Background Lock

Use `AppLifecycleListener` (Flutter 3.13+) or `WidgetsBindingObserver` to detect app going to background / returning to foreground:

- On `paused`: record a timestamp.
- On `resumed`: if time elapsed since `paused` ≥ idle timeout threshold, immediately show lock screen. Otherwise resume the idle timer from the remaining time.

OQ1 resolved — threshold-based: if background time ≥ idle threshold → lock immediately on foreground return. If background time < idle threshold → resume timer with the remaining time.

**Owner:** df-coder  
**Completion Criteria:** App locks on return from background if idle threshold was exceeded. Does not lock if returned quickly.

---

### Stage 7 — Settings Screen (Idle Timeout Config)

Replace the Phase 1 `SettingsScreen` placeholder with a functional settings screen:

- Single option visible in Phase 3: **Idle Timeout Duration**.
- Display current value (minutes). Allow selection from a predefined set: 1, 2, 5, 10, 15, 30 minutes (shown as a dropdown or radio group).
- Persist selection via `SettingsService` (SharedPreferences, key `'idle_timeout_minutes'`).
- `IdleTimeoutService` reads the updated value on next timer reset.
- `AsyncNotifierProvider` for `SettingsService` reads (async load from SharedPreferences); `NotifierProvider` is not appropriate here since loading is async.

**Owner:** df-coder  
**Completion Criteria:** Settings screen shows idle timeout option. Selection persists across restarts. `IdleTimeoutService` uses the saved value.

---

### Stage 8 — Unit Tests

Tests in `test/core/auth/`, `test/core/idle/`, `test/features/settings/`:

- `AuthService`: mock `LocalAuthentication`; test `authenticate` success, failure, `isDeviceSupported` false.
- `IdleTimeoutService`: timer fires after configured duration; `resetTimer` delays fire; `pauseTimer` cancels; `resumeTimer` restarts. Use `fake_async` (`^1.3.1`) for timer testing without real delays.
- `SettingsService`: read/write idle timeout via `SharedPreferences.setMockInitialValues` (from `shared_preferences` test helpers).

Add `fake_async` to `dev_dependencies`.

**Owner:** df-coder  
**Completion Criteria:** `flutter test` passes; all timer behaviour verified without real delays.

---

### Stage 9 — Validation

1. `dart format lib/ test/`
2. `flutter analyze` — zero issues
3. `flutter test` — all tests pass
4. `flutter build apk --debug` — succeeds
5. Manual smoke on emulator: cold launch → lock screen → authenticate → home → wait idle timeout → lock screen → re-authenticate → home → background for > timeout → foreground → lock screen.

**Owner:** df-coder  
**Completion Criteria:** All five checks pass. df-coder handoff includes output summary and emulator/device used.

---

## Progress Sheet

| Stage | Owner    | Status      | Completion Criteria                                                   | Notes                                          |
| ----- | -------- | ----------- | --------------------------------------------------------------------- | ---------------------------------------------- |
| 1 — Add Dependencies        | df-coder | Not Started | pubspec updated; pub get succeeds; AndroidManifest updated           | FlutterFragmentActivity required for local_auth |
| 2 — AuthService             | df-coder | Not Started | Interface + impl + auth provider compile cleanly                     |                                                |
| 3 — Lock Screen UI          | df-coder | Not Started | Authenticate works; failure handled; back blocked                    | Placeholder text for OQ2 is acceptable         |
| 4 — Cold Launch Lock        | df-coder | Not Started | App opens to lock screen; post-auth home; back doesn't return to lock |                                               |
| 5 — IdleTimeoutService      | df-coder | Not Started | Timer fires; reset/pause/resume work; Listener in root               | isLocked state → ref.listen in app.dart → navigate |
| 6 — Background Lock         | df-coder | Not Started | Background ≥ threshold → lock on resume                              | OQ1 resolved — threshold-based                 |
| 7 — Settings Screen         | df-coder | Not Started | Idle timeout option persists; service picks up new value             |                                                |
| 8 — Unit Tests              | df-coder | Not Started | flutter test passes; fake_async used for timer tests                 |                                                |
| 9 — Validation              | df-coder | Not Started | format/analyze/test/build/smoke all pass                             |                                                |

---

## Acceptance Criteria

1. `local_auth` and `shared_preferences` added to `pubspec.yaml` with no version conflicts.
2. `AndroidManifest.xml` has required biometric permissions and `FlutterFragmentActivity` as main activity.
3. `AuthService` is an abstract interface; `AuthServiceLocal` is the sole concrete implementation.
4. `authenticate` uses `biometricOnly: false` — PIN/pattern fallback is available.
5. Cold launch always shows lock screen before any health data or navigation is visible.
6. After successful authentication, the lock screen route is removed from the navigation stack (back does not return to lock screen).
7. Failed authentication shows a user-facing message and stays on the lock screen.
8. If `isDeviceSupported()` is false, the lock screen shows a clear message (placeholder text acceptable for Phase 3).
9. Idle timer resets on every user interaction (pointer-down).
10. Idle timer fires after the configured duration and navigates to lock screen from any screen.
11. Idle timer pauses while lock screen is visible; resets on successful authentication.
12. Background for ≥ idle threshold → lock screen on foreground return.
13. Settings screen shows idle timeout duration option. Selected value persists across app restarts.
14. `IdleTimeoutService` uses the value from SharedPreferences (default 5 min on first run).
15. `AuthService` methods have unit tests covering success, failure, and unavailable cases.
16. `IdleTimeoutService` timer behaviour is unit-tested using `fake_async`.
17. `SettingsService` read/write is unit-tested using SharedPreferences mock.
18. `flutter analyze` passes with zero issues.
19. `flutter test` passes with all tests green.
20. `flutter build apk --debug` succeeds.
21. Manual smoke test on emulator confirms full auth flow, idle timeout, and background lock.

---

## Risks and Mitigations

| Risk                                                                                 | Likelihood | Impact | Mitigation                                                                                   |
| ------------------------------------------------------------------------------------ | ---------- | ------ | -------------------------------------------------------------------------------------------- |
| `local_auth` requires `FlutterFragmentActivity`; Phase 1 may have used `FlutterActivity` | High    | High   | Stage 1 explicitly checks and replaces the activity class.                                  |
| Emulator may not support biometric auth testing                                      | Medium     | Medium | `local_auth` supports PIN fallback; use device PIN on emulator for smoke test.              |
| `AppLifecycleListener` requires Flutter ≥ 3.13; older SDK may not have it           | Low        | Low    | Fall back to `WidgetsBindingObserver` if not available. Note which is used in handoff.      |
| Idle timer and navigation state interacting unexpectedly during tests                | Medium     | Medium | Unit-test `IdleTimeoutService` in isolation with `fake_async`; integration tested via smoke. |

---

## Handoff Target

**df-qa** — Phase 3 plan revised (2026-06-02, cycle 2). M4 applied: `GlobalKey<NavigatorState> _navigatorKey` introduced in `app.dart` state; passed to `MaterialApp(navigatorKey:)`; `ref.listen` handler updated to use `_navigatorKey.currentState?.pushNamedAndRemoveUntil(...)`. L3 applied: stale OQ1 risk row removed. All other findings remain resolved.
