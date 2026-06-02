# df-qa to df-coder

## Summary

**Phases 1–7** are all approved. Implement phases in order (1 → 2 → 3 → 4 → 5 → 6 → 7), routing back to df-qa after each phase before proceeding to the next. This handoff covers all seven phases.

---

## Current Stage and Status

- Stage: **Active — implementation queue open; Phases 1–7 approved**
- Active routing: df-qa → df-coder (this handoff)
- Phase 1 detailed plan status: **Approved — Ready for Implementation**
- Phase 2 detailed plan status: **Approved — Ready for Implementation (after Phase 1 QA pass)**
- Phase 3 detailed plan status: **Approved — Ready for Implementation (after Phases 1 + 2)**
- Phase 4 detailed plan status: **Approved — Ready for Implementation (after Phases 1–3)**
- Phase 5 detailed plan status: **Approved — Ready for Implementation (after Phase 4)**
- Phase 6 detailed plan status: **Approved — Ready for Implementation (after Phase 5)**
- Phase 7 detailed plan status: **Approved — Ready for Implementation (after Phase 6)**
- Active plans:
  - `Documentation/Plans/Plan_Phase1_Scaffold.md`
  - `Documentation/Plans/Plan_Phase2_Data_Model.md`
  - `Documentation/Plans/Plan_Phase3_Security_Layer.md`
  - `Documentation/Plans/Plan_Phase4_Symptom_Logging.md`
  - `Documentation/Plans/Plan_Phase5_History_Review.md`
  - `Documentation/Plans/Plan_Phase6_Printable_Reports.md`
  - `Documentation/Plans/Plan_Phase7_Polish_Release.md`
  - `Documentation/Plans/Plan_Initial_App_Delivery.md` (parent)

---

## Context and Constraints

- Android-first Flutter application. iOS is not in scope for v1.
- Local-first, offline-first, single-user. No backend or sync.
- All workflow artifacts must remain inside this repository under `Documentation/`.
- Phase 1 scope is scaffold only — no feature logic, no auth logic, no storage, no data models.
- Rapid delivery is a product priority. Keep implementation tight.

---

## Implementation Scope

Follow `Plan_Phase1_Scaffold.md` stages in order. All decisions are finalized — do not deviate without raising a new handoff.

### Stage 1 — Flutter Project Creation

**Critical pre-step:** Preserve `README.md` before running `flutter create`:

```sh
copy README.md README.md.bak
flutter create --org org.zephon --project-name symptom_diary .
del README.md
rename README.md.bak README.md
```

Verify after creation:

- `pubspec.yaml` name: `symptom_diary`
- `android/app/build.gradle` applicationId: `org.zephon.symptom_diary`

### Stage 2 — Directory Structure

Create the full `lib/` feature layout including `lib/features/shared/providers/`. Every `_screen.dart` is a minimal stateless placeholder only. `navigation_provider.dart` is a stub comment file in Stage 2; its content is written in Stage 5.

Full target structure is in `Plan_Phase1_Scaffold.md` § Stage 2.

### Stage 3 — Navigation Shell

Bottom nav bar (4 tabs: Home, History, Reports, Settings) + `/lock` route.

**Sequencing note (LOW — informational only):** Stage 3 uses `ConsumerStatefulWidget` and `navigationIndexProvider`, which require `flutter_riverpod`. Add `flutter_riverpod` to `pubspec.yaml` and run `flutter pub get` at the start of Stage 3 — do not wait until Stage 5 for the package addition. Stage 5 is responsible for defining the provider content; Stage 3 can add the package dependency.

`MainShell` must be a `ConsumerStatefulWidget`. Named routes: `/` → `MainShell`, `/lock` → `LockScreen`.

### Stage 4 — App Theming

`AppTheme.lightTheme` and `AppTheme.darkTheme` as static `ThemeData` getters in `core/theme/app_theme.dart`. M3 enabled. Seed color `Color(0xFF1565C0)`. `ThemeMode.system`.

### Stage 5 — Riverpod Scaffold

Define `navigationIndexProvider` (`StateProvider<int>`) in `lib/features/shared/providers/navigation_provider.dart`. Wrap app with `ProviderScope` in `main.dart`. No other providers created.

`StateProvider<int>` is an approved exception for Phase 1 scaffold (see plan note in Stage 5). Do not create `NotifierProvider` variants at this stage.

### Stage 6 — Android SDK Configuration

Confirm `android/app/build.gradle`:

- `compileSdkVersion 31`
- `minSdkVersion 23`
- `targetSdkVersion 31`
- `applicationId` = `org.zephon.symptom_diary`

Verify only — do not change unless values are wrong.

### Stage 7 — Validation

Run all four checks and record results:

| Check | Command | Required Outcome |
| --- | --- | --- |
| Formatter | `dart format lib/` | No changes |
| Analyzer | `flutter analyze` | Zero issues |
| Debug build | `flutter build apk --debug` | Completes without errors |
| Manual smoke | Run on emulator | Launches; all 4 tabs navigate; no console errors |

Document the emulator/device used. All four checks must pass before routing to df-qa.

---

## Acceptance Criteria to Verify Before Routing Back

Check each item and report pass/fail in the return handoff:

1. `pubspec.yaml` `name` = `symptom_diary`; `applicationId` = `org.zephon.symptom_diary`
2. Directory structure matches Stage 2 layout exactly (including `shared/providers/`)
3. App launches on Android without a runtime error
4. Bottom nav has exactly 4 tabs: Home, History, Reports, Settings
5. Each tab shows a named placeholder screen
6. `/lock` route renders `LockScreen` placeholder
7. `ProviderScope` wraps `MaterialApp`
8. `navigationIndexProvider` is the only provider defined and is consumed by `MainShell`
9. `AppTheme` applied; M3 enabled; dark/light responds to OS toggle
10. Android SDK: `minSdkVersion 23`, `targetSdkVersion 31`, `compileSdkVersion 31`
11. `dart format lib/` — no changes needed
12. `flutter analyze` — zero issues
13. `flutter build apk --debug` — no errors

---

## Return Handoff Requirements

When all 7 stages are complete and all acceptance criteria pass, create or update `Documentation/Handoffs/df-coder_to_df-qa.md` with:

- Stage completion status for each of the 7 stages
- Validation results table (pass/fail for each Stage 7 check, with emulator/device identified)
- Any deviations from the plan (if none, state explicitly)
- Any issues encountered and how they were resolved
- Any items that could not be completed (must list reason)

Do not route to df-qa until all Stage 7 checks pass. If a check cannot be run (e.g., no Android emulator available), state this explicitly and document the reason.

---

# Phase 2 — Data Model and Encrypted Persistence

**Plan:** `Documentation/Plans/Plan_Phase2_Data_Model.md`  
**Prerequisite:** Phase 1 complete and approved by df-qa.  
**All 8 stages: Not Started**

---

## Phase 2 Context and Constraints

- All health data stored in encrypted Hive boxes. No plaintext in any box file.
- Encryption key in `flutter_secure_storage` only — never in source, logs, or box files.
- Services use constructor injection for their Hive `Box<T>`. No `Hive.box(...)` calls inside service methods.
- Predefined symptom list is a `[]` stub. Content is Phase 4.
- Providers created here are stubs only. Full UI wiring is Phase 4.

---

## Phase 2 Implementation Scope

Follow `Plan_Phase2_Data_Model.md` stages in order. Key decisions are finalized.

### Stage 1 — Add Dependencies

Add to `pubspec.yaml` and run `flutter pub get`:

```yaml
dependencies:
  hive: ^2.2.3
  hive_flutter: ^1.1.0
  flutter_secure_storage: ^9.2.4
  uuid: ^4.5.1

dev_dependencies:
  hive_generator: ^2.0.1
  build_runner: ^2.4.0
```

Before running `flutter pub get`: check the `flutter_secure_storage` package documentation for any required `AndroidManifest.xml` entries at the pinned version.

### Stage 2 — Typed Models

Create `DailyEntry` (typeId 0), `Symptom` (typeId 1), `Preset` (typeId 2), and `PainLevel` (utility, no Hive annotation) in their feature directories.  
`DailyEntry.date` is **always** normalized to midnight local: `DateTime(d.year, d.month, d.day)`.

### Stage 3 — Code Generation

```
dart run build_runner build --delete-conflicting-outputs
```
Commit all generated `.g.dart` files.

### Stage 4 — Encrypted Storage Initialisation

Key management in `lib/core/storage/encrypted_storage.dart`:
- First run: generate 32-byte AES key → `flutter_secure_storage`.
- Subsequent runs: retrieve key from `flutter_secure_storage`.
- Key mismatch (decrypt failure, e.g. post-reinstall): catch exception → delete corrupted boxes → reinitialise fresh → log event.
- Open boxes: `diary_entries`, `symptoms`, `presets`. Return `Box<T>` references for injection.
- `EncryptedStorage.init()` called before `runApp` in `main.dart`.

### Stage 5 — DiaryService

`DiaryServiceHive(Box<DailyEntry> box)` — box injected, never fetched internally.

Box key for `diary_entries`: normalized date string `entry.date.toIso8601String().substring(0, 10)`.

**Implementation note (N2):** `getEntryForDate(DateTime date)` must normalize the incoming `date` parameter before computing the key: `final key = DateTime(date.year, date.month, date.day).toIso8601String().substring(0, 10);` then `box.get(key)`. A non-normalized incoming date would produce a cache miss.

### Stage 6 — SymptomService and PresetService

`SymptomServiceHive(Box<Symptom> box)` and `PresetServiceHive(Box<Preset> box)` — both constructor-injected.

**Implementation note (N3):** Add `lib/features/symptoms/providers/presets_provider.dart` with a stub `AsyncNotifierProvider<PresetsNotifier, List<Preset>>`. This was omitted from the plan; apply during Stage 6.

Stub providers for both DiaryService and SymptomService are already in the plan. Add Preset's provider alongside them.

### Stage 7 — Unit Tests

All service tests use constructor injection:
```dart
setUp(() async {
  tempDir = await Directory.systemTemp.createTemp();
  Hive.init(tempDir.path);
  if (!Hive.isAdapterRegistered(0)) Hive.registerAdapter(DailyEntryAdapter());
  if (!Hive.isAdapterRegistered(1)) Hive.registerAdapter(SymptomAdapter());
  if (!Hive.isAdapterRegistered(2)) Hive.registerAdapter(PresetAdapter());
  entryBox = await Hive.openBox<DailyEntry>('diary_entries');
  // (open symptom/preset boxes similarly for their respective tests)
});
tearDown(() async {
  await Hive.deleteFromDisk();
  tempDir.deleteSync(recursive: true);
});
```

**Implementation note (N1):** The plan's Stage 7 still contains a stale "If OQ1 Yes:" qualifier on the Preset tests. Disregard the conditional — Preset is unconditional. Implement all Preset adapter and service tests.

### Stage 8 — Validation

1. `dart format lib/ test/`
2. `flutter analyze` — zero issues
3. `flutter test` — all tests pass
4. `flutter build apk --debug` — succeeds

---

## Phase 2 Acceptance Criteria (summary — full list in plan AC 1–18)

- All 18 AC items in `Plan_Phase2_Data_Model.md` must pass.
- Key items: encrypted Hive boxes; key in `flutter_secure_storage` only; constructor injection on all services; upsert-by-date verified by test; `getEntryForDate` normalizes incoming date; provider stubs for Diary, Symptoms, and Presets; 18 AC checks all green.

---

## Phase 2 Return Handoff Requirements

When all 8 stages complete and all AC pass, update `Documentation/Handoffs/df-coder_to_df-qa.md` with:

- Stage completion status for all 8 stages
- Validation output summary (format, analyze, test, build)
- Confirmation that N1/N2/N3 implementation notes were applied
- Any deviations from the plan (if none, state explicitly)


---

## Files to Create or Modify

| File | Action | Stage |
| --- | --- | --- |
| `README.md` | Preserve before `flutter create`, restore after | Stage 1 |
| `pubspec.yaml` | Add `flutter_riverpod: ^2.6.0` | Stage 3 (package) / Stage 5 (provider) |
| `android/app/build.gradle` | Verify SDK values | Stage 6 |
| `lib/main.dart` | `ProviderScope` wrapping `runApp(MaterialApp(...))` | Stage 5 |
| `lib/app.dart` | `MaterialApp` with theme and named routes | Stage 3 |
| `lib/core/theme/app_theme.dart` | `AppTheme.lightTheme` and `AppTheme.darkTheme` | Stage 4 |
| `lib/features/auth/screens/lock_screen.dart` | Placeholder widget | Stage 2 |
| `lib/features/diary/screens/home_screen.dart` | Placeholder widget | Stage 2 |
| `lib/features/diary/screens/log_entry_screen.dart` | Placeholder widget | Stage 2 |
| `lib/features/history/screens/history_screen.dart` | Placeholder widget | Stage 2 |
| `lib/features/reports/screens/report_screen.dart` | Placeholder widget | Stage 2 |
| `lib/features/settings/screens/settings_screen.dart` | Placeholder widget | Stage 2 |
| `lib/features/shared/providers/navigation_provider.dart` | Stub in Stage 2; `navigationIndexProvider` in Stage 5 | Stages 2 and 5 |

`MainShell` widget lives in `lib/app.dart` or a dedicated `lib/features/shared/widgets/main_shell.dart` — df-coder's choice, provided it follows snake_case file naming.

---

# Phase 3 — Security Layer

**Plan:** `Documentation/Plans/Plan_Phase3_Security_Layer.md`  
**Prerequisite:** Phase 1 **and** Phase 2 complete and approved by df-qa.  
**All 9 stages: Not Started**

---

## Phase 3 Context and Constraints

- Authentication via `local_auth` — biometric or device PIN/pattern. No custom PIN management.
- Lock screen shown on cold launch, idle timeout, and background return after threshold.
- All health data remains inaccessible until authentication passes.
- Idle timeout and settings use `shared_preferences` (unencrypted — no health data).
- `FlutterFragmentActivity` is required by `local_auth`. Stage 1 verifies and replaces if needed. This is high-risk — do not skip.
- Android-first; iOS out of scope for v1.

---

## Phase 3 Implementation Scope

Follow `Plan_Phase3_Security_Layer.md` stages in order. All decisions are finalized.

### Stage 1 — Add Dependencies

Add to `pubspec.yaml`:

```yaml
dependencies:
  local_auth: ^2.3.0
  shared_preferences: ^2.3.0

dev_dependencies:
  fake_async: ^1.3.1
```

`AndroidManifest.xml` additions:

```xml
<uses-permission android:name="android.permission.USE_BIOMETRIC"/>
<uses-permission android:name="android.permission.USE_FINGERPRINT"/>
```

Verify main activity is `FlutterFragmentActivity`. If `FlutterActivity` is present, replace it.

### Stage 2 — AuthService

Abstract interface (`authenticate`, `isDeviceSupported` only):

```dart
abstract class AuthService {
  Future<bool> authenticate({required String reason});
  Future<bool> isDeviceSupported();
}
```

`canCheckBiometrics` is deliberately excluded — it tests biometric hardware only. `isDeviceSupported()` is the credential-availability guard (returns `false` if no PIN, pattern, or biometric is enrolled).

`AuthState` enum: `locked`, `authenticated`, `unavailable`. `AsyncNotifierProvider<AuthNotifier, AuthState>` in `lib/features/auth/providers/auth_provider.dart`.

### Stage 3 — Lock Screen UI

Replace Phase 1 placeholder. `PopScope(canPop: false, onPopInvokedWithResult: (_, __) {})` is the sole back-nav blocker. `WillPopScope` must not be used (removed in Flutter 3.24+).

Success path: `authNotifier.authenticate()` → state becomes `authenticated` → `Navigator.pushNamedAndRemoveUntil('/', (_) => false)`.

If `isDeviceSupported()` is false: replace Unlock button with a message (placeholder text acceptable).

### Stage 4 — Cold Launch Lock

Initial route: `/lock` unconditionally in `app.dart`. `EncryptedStorage.init()` (Phase 2) is called before `runApp`; boxes open but no data shown until authentication passes.

### Stage 5 — IdleTimeoutService

Key implementation pattern — `GlobalKey<NavigatorState>` in `app.dart` state:

```dart
// In _AppState:
final GlobalKey<NavigatorState> _navigatorKey = GlobalKey<NavigatorState>();

// In build() — ref.listen fires navigation when isLocked becomes true:
ref.listen<IdleTimeoutState>(idleTimeoutProvider, (prev, next) {
  if (next.isLocked && !(prev?.isLocked ?? false)) {
    ref.read(authProvider.notifier).lock();
    _navigatorKey.currentState?.pushNamedAndRemoveUntil('/lock', (_) => false);
  }
});

// Root widget tree:
Listener(
  behavior: HitTestBehavior.translucent,
  onPointerDown: (_) => ref.read(idleTimeoutProvider.notifier).resetTimer(),
  child: MaterialApp(
    navigatorKey: _navigatorKey,
    ...
  ),
)
```

**Implementation note (L4 — LOW):** Stage 5 in the plan contains two `Listener`/`MaterialApp` code blocks. Use the **second** block (which includes `navigatorKey: _navigatorKey`) as the definitive implementation. The first block is a superseded excerpt without the key — ignore it.

**Implementation note (L5 — LOW):** Stage 6 background lock says "immediately show lock screen." For consistency with the Stage 5 idle-timeout pattern, trigger it by setting `isLocked = true` in `IdleTimeoutState` from the `AppLifecycleListener`/`WidgetsBindingObserver` handler. The same `ref.listen` in `app.dart` will handle navigation automatically. Alternatively, call `_navigatorKey.currentState?.pushNamedAndRemoveUntil(...)` directly from the lifecycle handler — both approaches are in `_AppState` scope and both work.

`IdleTimeoutState` (immutable): `{ bool isLocked, Duration timeout }`. Timer fires → `isLocked = true`, timer cancelled. `pauseTimer()` cancels without restarting. `resumeTimer()` restarts and resets `isLocked` to `false`.

### Stage 6 — Background Lock

`AppLifecycleListener` (Flutter 3.13+) or `WidgetsBindingObserver` in `app.dart`:
- `paused`: record timestamp.
- `resumed`: elapsed ≥ threshold → lock (via `isLocked = true` or direct `_navigatorKey` call — see L5 note above). elapsed < threshold → resume timer with remaining time.

### Stage 7 — Settings Screen

`AsyncNotifierProvider` for `SettingsService` (SharedPreferences load is async). Predefined duration options: 1, 2, 5, 10, 15, 30 minutes. SharedPreferences key: `'idle_timeout_minutes'`. Default: 5.

### Stage 8 — Unit Tests

- `AuthService`: mock `LocalAuthentication`; test success, failure, `isDeviceSupported` false.
- `IdleTimeoutService`: use `fake_async` for all timer tests.
- `SettingsService`: `SharedPreferences.setMockInitialValues` for read/write tests.

### Stage 9 — Validation

1. `dart format lib/ test/`
2. `flutter analyze` — zero issues
3. `flutter test` — all tests pass
4. `flutter build apk --debug` — succeeds
5. Manual smoke on emulator: cold launch → lock → authenticate → home → idle timeout → lock → re-authenticate → background > timeout → foreground → lock.

---

## Phase 3 Acceptance Criteria (summary — full list in plan AC 1–21)

All 21 AC items in `Plan_Phase3_Security_Layer.md` must pass. Key items: `FlutterFragmentActivity` set; auth interface 2 methods only; cold launch shows lock; idle timer fires via `isLocked → ref.listen → navigate`; background threshold lock; settings persist; `fake_async` timer tests; format/analyze/test/build/smoke all green.

---

## Phase 3 Return Handoff Requirements

When all 9 stages complete and all 21 AC pass, update `Documentation/Handoffs/df-coder_to_df-qa.md` with:

- Stage completion status for all 9 stages
- Validation output summary (format, analyze, test, build, smoke)
- Emulator/device used for smoke test
- Confirmation that L4 and L5 implementation notes were applied
- Any deviations from the plan (if none, state explicitly)

---

---

# Phase 4 — Symptom Logging

**Plan:** `Documentation/Plans/Plan_Phase4_Symptom_Logging.md`  
**Prerequisite:** Phase 1, Phase 2, and Phase 3 complete and approved by df-qa.  
**All 9 stages: Not Started**

---

## Phase 4 Context and Constraints

- Replace Phase 2 stub providers with real `AsyncNotifier` implementations. No direct Hive calls in UI code.
- Replace Phase 1 placeholder screens with real UI: `HomeScreen`, `LogEntryScreen`, `AddSymptomScreen`, `PresetsScreen`.
- Phase 2's `EncryptedStorage.init()` already opens all Hive boxes before `runApp`. Those `Box<T>` references must enter the Riverpod graph via `ProviderScope` overrides — never via `Hive.box(...)` calls inside notifiers.
- Future dates (beyond today) are not permitted anywhere in the entry flow.
- OQ1 (predefined symptom list) is open and non-blocking — use the placeholder `kPredefinedSymptoms` constant from the plan. Note OQ1 status in the return handoff.
- OQ2 (home screen entry display) is open and non-blocking — default: pain level + symptom summary if entry exists, "Log Today" CTA if not.
- Rapid delivery is a product priority. Keep implementation tight.

---

## Phase 4 Implementation Scope

Follow `Plan_Phase4_Symptom_Logging.md` stages in order. All decisions are finalized.

### Stage 1 — Wire Providers

**Box providers and ProviderScope bootstrapping:**

```dart
final diaryBoxProvider = Provider<Box<DailyEntry>>(
  (ref) => throw UnimplementedError('not initialized'));
final symptomBoxProvider = Provider<Box<Symptom>>(
  (ref) => throw UnimplementedError('not initialized'));
final presetBoxProvider = Provider<Box<Preset>>(
  (ref) => throw UnimplementedError('not initialized'));
```

In `main.dart`, after `EncryptedStorage.init()`:

```dart
final boxes = await EncryptedStorage.init();
runApp(
  ProviderScope(
    overrides: [
      diaryBoxProvider.overrideWithValue(boxes.diaryBox),
      symptomBoxProvider.overrideWithValue(boxes.symptomBox),
      presetBoxProvider.overrideWithValue(boxes.presetBox),
    ],
    child: MyApp(),
  ),
);
```

Service providers read their box via `ref.watch`:

```dart
final diaryServiceProvider = Provider<DiaryServiceHive>(
  (ref) => DiaryServiceHive(ref.watch(diaryBoxProvider)));
```

Apply the same pattern for `symptomServiceProvider` and `presetServiceProvider`.

**DiaryNotifier — family provider:**

- Type: `AsyncNotifierProvider.family<DiaryNotifier, DailyEntry?, DateTime>`
- `build(DateTime date)` loads and returns `DailyEntry?` automatically. No `loadForDate` method.
- Methods: `saveEntry(DailyEntry)`, `deleteEntry(DateTime)`.
- After `saveEntry`: `state = AsyncData(entry)`. Do not use `ref.invalidate`.
- Call sites: `ref.watch(diaryProvider(date))` to observe; `ref.read(diaryProvider(date).notifier)` to mutate.

**SymptomsNotifier:**

- Type: `AsyncNotifierProvider<SymptomsNotifier, List<Symptom>>`
- `build()` merges `kPredefinedSymptoms` + Hive custom: `return [...kPredefinedSymptoms, ...custom]`
- Methods: `addCustom(String name)`, `deleteCustom(String id)`.
- After `addCustom` or `deleteCustom`: update `state = AsyncData(newList)`. See **Implementation note N1**.

**PresetsNotifier:**

- Type: `AsyncNotifierProvider<PresetsNotifier, List<Preset>>`
- Methods: `savePreset(String name, List<String> symptomIds)`, `deletePreset(String id)`.
- After mutations: `state = AsyncData(newList)`.

### Stage 2 — Home Screen

- Watches `diaryProvider(today)` where `today = DateTime(now.year, now.month, now.day)`.
- If entry exists: show pain level badge and up to 3 symptom chips. Resolve `entry.symptomIds` to names via `ref.watch(symptomsProvider)`: `allSymptoms.where((s) => entry.symptomIds.contains(s.id))`.
- If no entry: show "Log Today" CTA.
- "Log Past Entry" text button opens a `showDatePicker` (restricted to dates ≤ today) then navigates to `LogEntryScreen`.

### Stage 3 — Log Entry Screen — Pain Level and Date

- Single screen for both create and edit. Accepts `DateTime date` parameter.
- Watches `diaryProvider(date)` — pre-populates from existing entry if present.
- `Slider(min: 0, max: 10, divisions: 10)` for pain level. Defaults to 0 if untouched.
- Date change icon is only enabled when launched from the "Log Past Entry" flow.

### Stage 4 — Symptom Picker

- `SymptomChipGrid` watches `ref.watch(symptomsProvider)` — renders the full combined list (predefined + custom) as `FilterChip` widgets.
- Chip selection stores `Symptom.id` values into local `selectedSymptomIds` form state.
- "Add symptom" chip at the end navigates to `AddSymptomScreen`. On save: calls `symptomsNotifier.addCustom(name)` then pops. The new chip appears immediately because `addCustom` updates `state` (N1).
- `kPredefinedSymptoms` constant in `lib/features/symptoms/symptom_data.dart`. Use the placeholder list from the plan.

### Stage 5 — Notes Field and Save Flow

- Multiline `TextField` for `DailyEntry.notes`.
- "Save" calls `diaryNotifier.saveEntry(entry)`, then `Navigator.pop()`.
- After save: notifier sets `state = AsyncData(entry)`. Home screen updates automatically via `ref.watch(diaryProvider(today))`. No `ref.invalidate` call needed.

### Stage 6 — Preset Management

- "Save as Preset" action in `LogEntryScreen` app bar. Dialog with name field. Calls `presetsNotifier.savePreset(name, selectedSymptomIds)`.
- `PresetsScreen`: watches both `presetsProvider` AND `symptomsProvider`. Resolve each preset's `symptomIds` to display names: `allSymptoms.where((s) => preset.symptomIds.contains(s.id)).map((s) => s.name)`.
- Register `/presets` as a named route in `app.dart`. Navigate via `Navigator.pushNamed(context, '/presets')`.
- Empty state when no presets exist.

### Stage 7 — Apply Preset Flow

- "Apply Preset" button on `LogEntryScreen` opens a bottom sheet or dialog listing presets from `ref.watch(presetsProvider)`.
- Tapping a preset: filter `preset.symptomIds` through `allSymptoms.map((s) => s.id).toSet()` to skip deleted custom symptoms (see **N2**), then replace local `selectedSymptomIds` state.

### Stage 8 — Provider and Widget Tests

- `DiaryNotifier`: load null (no entry); save creates entry; save same date updates (no duplicate); delete removes.
- `SymptomsNotifier`: `build()` returns `[...kPredefinedSymptoms, ...custom]`; `addCustom` persists and updates state; `deleteCustom` removes and updates state; predefined list not in Hive.
- `PresetsNotifier`: save persists; delete removes.
- Widget test `LogEntryScreen`: loads existing entry; save calls notifier; back nav from save.
- Widget test `HomeScreen`:
  - Shows CTA when `diaryProvider(today)` returns null.
  - Shows summary (AC9) — override **both** `diaryProvider(today)` with a `DailyEntry` containing `symptomIds` **and** `symptomsProvider` with a matching `List<Symptom>`.
- All tests use `ProviderScope(overrides: [...])` with mock services.
- `DiaryNotifier` tests: `container.read(diaryProvider(date).notifier)`.
- Apply preset behaviour: implement as a widget test (see **N2**).

### Stage 9 — Validation

1. `dart format lib/ test/`
2. `flutter analyze` — zero issues
3. `flutter test` — all tests pass
4. `flutter build apk --debug` — succeeds
5. Manual smoke: open app → authenticate → home (no entry) → log today → pain + symptoms + notes → save → home reflects entry → tap entry → edit → save → log past entry → preset save → preset apply → presets screen delete.

---

## Phase 4 Implementation Notes

**N1 — `SymptomsNotifier` state update after mutations:**

After `addCustom` or `deleteCustom`, update `state` directly — do not rely on navigation pop to trigger a rebuild. Example for `addCustom`:

```dart
Future<void> addCustom(String name) async {
  final service = ref.read(symptomServiceProvider);
  final newSymptom = Symptom(id: const Uuid().v4(), name: name, isPredefined: false);
  await service.saveSymptom(newSymptom);
  final current = state.requireValue;
  state = AsyncData([...current, newSymptom]);
}
```

This ensures `SymptomChipGrid`'s `ref.watch(symptomsProvider)` picks up the new symptom immediately when `AddSymptomScreen` pops back (AC5).

**N2 — Apply preset: filter deleted custom symptoms; "apply" test is a widget test:**

When applying a preset in Stage 7, filter through the current `allSymptoms` list:

```dart
final allIds = allSymptoms.map((s) => s.id).toSet();
final validIds = preset.symptomIds.where((id) => allIds.contains(id)).toList();
// Replace local selectedSymptomIds with validIds
```

The Stage 8 PresetsNotifier test line "apply returns correct symptom list" refers to this filtering behaviour. There is no `apply` method on `PresetsNotifier` (Stage 1 defines only `savePreset` and `deletePreset`). Implement the apply test as a widget test: tap "Apply Preset" → verify chip selection updates to the filtered list. This also covers AC13 and AC15.

---

## Phase 4 Acceptance Criteria (summary — full list in plan AC 1–21)

All 21 AC items in `Plan_Phase4_Symptom_Logging.md` must pass. Key items: pain slider interactive (AC1); defaults to 0 (AC2); pre-populates existing entry (AC3); chips show predefined + custom (AC4); custom symptom appears immediately (AC5); notes saves multiline (AC6); no duplicate entries (AC7); home CTA when no entry (AC8); home summary when entry exists (AC9); date picker restricted to past dates (AC10–11); preset save/apply/delete (AC12–14); missing symptom skipped (AC15); unit tests (AC16–17); format/analyze/test/build/smoke (AC18–21).

---

## Phase 4 Return Handoff Requirements

When all 9 stages complete and all 21 AC pass, update `Documentation/Handoffs/df-coder_to_df-qa.md` with:

- Stage completion status for all 9 stages
- Validation output summary (format, analyze, test, build, smoke)
- Emulator/device used for smoke test
- Confirmation that N1 and N2 implementation notes were applied
- OQ1 status (placeholder list used if unresolved — state explicitly)
- Any deviations from the plan (if none, state explicitly)

---

---

# Phase 5 — History Review

**Plan:** `Documentation/Plans/Plan_Phase5_History_Review.md`  
**Prerequisite:** Phases 1–4 complete and approved by df-qa.  
**All 6 stages: Not Started**

---

## Phase 5 Context and Constraints

- Replace `HistoryScreen` placeholder (Phase 1) with a real calendar-based history view.
- All data reads go through `DiaryServiceHive` and `SymptomsNotifier` (Phase 4). No direct Hive calls in UI or provider code.
- One new package dependency: `table_calendar: ^3.1.0`. Compatible with minSdkVersion 23 / compileSdkVersion 31.
- OQ1 (`table_calendar` vs custom grid) is open and non-blocking — default `table_calendar` implementation applies. Note OQ1 status in return handoff.
- OQ2 (future day tap) is open and non-blocking — default: no action on future days.
- Rapid delivery is a product priority.

---

## Phase 5 Implementation Scope

Follow `Plan_Phase5_History_Review.md` stages in order. All decisions are finalised.

### Stage 1 — Wire History Provider

- `HistoryNotifier`: `AsyncNotifierProvider<HistoryNotifier, List<DailyEntry>>` (non-family). `build()` calls `ref.watch(diaryServiceProvider).getAllEntries()`. If `getAllEntries()` is absent from `DiaryServiceHive`, add it: `box.values.toList()` sorted by `entry.date` descending.
- `HistoryFilterNotifier`: `NotifierProvider<HistoryFilterNotifier, HistoryFilterState>`. `build()` returns `const HistoryFilterState()`. Methods: `setDateRange(DateTimeRange?)`, `setSymptomFilter(List<String>)`, `setPainThreshold(int?)`, `clearAll()`.
- `HistoryFilterState`: immutable class with `dateRange`, `symptomIds`, `painThreshold`. `const` constructor. `copyWith()`.
- `_applyFilter`: pure function (not a provider). Called inside `HistoryScreen` widget: `final filtered = _applyFilter(allEntries, filter)`.
- State can be invalidated: `ref.invalidate(historyProvider)` re-runs `build()` and reloads from Hive.

### Stage 2 — History Screen (Calendar View)

- Add `table_calendar: ^3.1.0` to `pubspec.yaml`; run `flutter pub get`.
- `TableCalendar` configured with `firstDay` (earliest entry date or 2-year floor), `lastDay` (today normalised), `focusedDay` (today), `calendarFormat` (month), `eventLoader` (pre-built `Map<DateTime, DailyEntry>` keyed by normalised date), `calendarBuilders.markerBuilder` (renders `EntryDayMarker`).
- `EntryDayMarker`: pain level colour dot (0–3 green, 4–6 amber, 7–10 red) + symptom presence dot.
- Day tap routing: entry day → `Navigator.push(EntryDetailScreen(date: day))`; empty past day → `Navigator.push(LogEntryScreen(date: day))`; future day → no action.
- Filter button in app bar opens `FilterSheet` bottom sheet.
- Wrap calendar in `RefreshIndicator`; `onRefresh` reloads from Hive:

  ```dart
  onRefresh: () async {
    ref.invalidate(historyProvider);
    await ref.read(historyProvider.future);
  },
  ```

  **Implementation note N1:** `RefreshIndicator.onRefresh` is typed as `Future<void> Function()`. Without `await ref.read(historyProvider.future)`, the spinner dismisses before reloaded data is available. Both lines are required for correct UX.

### Stage 3 — Entry Detail Screen

- New screen at `lib/features/history/screens/entry_detail_screen.dart`. Accepts `DateTime date`.
- Watches `diaryProvider(date)` (Phase 4 family provider). Renders: formatted date, pain level (numeric + indicator), symptom chips (names resolved via `ref.watch(symptomsProvider)`), notes.
- App bar "Edit" button → `LogEntryScreen(date: date)`.
- Null guard: if `diaryProvider(date)` returns null, show "No entry for this date" + back button.
- Register `/entry_detail` named route in `app.dart`. Navigate via `Navigator.pushNamed(context, '/entry_detail', arguments: date)`. Extract `DateTime` from `RouteSettings.arguments` in the route builder.

### Stage 4 — Filter and Sort

- `FilterSheet`: `DraggableScrollableSheet` modal bottom sheet.
  1. **Date range**: `showDateRangePicker`; calls `notifier.setDateRange(range)`.
  2. **Symptom filter**: `FilterChip` grid from `ref.watch(symptomsProvider)`; calls `notifier.setSymptomFilter(selectedIds)`.
  3. **Pain threshold**: `Slider(min: 0, max: 10, divisions: 10)`; `onChanged` must cast: `notifier.setPainThreshold(value.toInt())`. (`divisions: 10` makes `toInt()` lossless.)
  4. **Clear all**: calls `notifier.clearAll()`.
- Active-filter indicator on filter button when any filter is non-default.
- `_applyFilter` logic: date range exclusion; symptom OR logic (empty = show all); pain threshold minimum (null = show all).
- Empty state: "No entries match your filters" + "Clear filters" text button.

### Stage 5 — Tests

- `HistoryNotifier`: build() returns all entries sorted descending; returns empty list when box empty.
- `HistoryNotifier` invalidation reload: seed box with one entry, load provider, add second entry directly to box, `ref.invalidate(historyProvider)`, assert both entries present.
- `DiaryServiceHive.getAllEntries()`: returns all entries; returns empty list when box empty.
- `HistoryFilterNotifier`: each mutation method produces correct state.
- `_applyFilter` pure function: date range, symptom (OR logic), pain threshold, combination of all three, no filters returns all.
- Widget test `HistoryScreen`: empty state when no entries; calendar markers when entries present.
- Widget test `EntryDetailScreen`: all fields render; Edit navigates to `LogEntryScreen`; null guard renders correctly.
- All provider tests use `ProviderScope(overrides: [...])` with mock services and box overrides.

### Stage 6 — Validation

1. `dart format lib/ test/`
2. `flutter analyze` — zero issues
3. `flutter test` — all tests pass
4. `flutter build apk --debug` — succeeds
5. Manual smoke: History tab → calendar → navigate months → tap entry day → detail → edit → back → tap empty past day → log entry → save → return to History → pull to refresh → verify new marker → filter sheet (date range, symptom, threshold) → clear filters.

---

## Phase 5 Acceptance Criteria (summary — full list in plan AC 1–22)

All 22 AC items in `Plan_Phase5_History_Review.md` must pass. Key items: monthly calendar with pain-level markers (AC1–2); no marker on empty day (AC3); entry day tap opens detail (AC4); detail shows all fields (AC5); empty past day tap opens log screen (AC6); future days not tappable (AC7); month navigation works (AC8); all three filter types work (AC9–12); empty filter state (AC12); empty state message + clear action (AC13); active filter indicator (AC14); unit tests (AC15–17); format/analyze/test/build (AC18–20); smoke (AC21); pull-to-refresh (AC22).

---

## Phase 5 Return Handoff Requirements

When all 6 stages complete and all 22 AC pass, update `Documentation/Handoffs/df-coder_to_df-qa.md` with:

- Stage completion status for all 6 stages
- Validation output summary (format, analyze, test, build, smoke)
- Emulator/device used for smoke test
- Confirmation that N1 (onRefresh async pattern) was applied
- OQ1 status (note which implementation was used)
- Any deviations from the plan (if none, state explicitly)

---

---

# Phase 6 — Printable Reports

**Plan:** `Documentation/Plans/Plan_Phase6_Printable_Reports.md`  
**Prerequisite:** Phases 1–5 complete and approved by df-qa.  
**All 5 stages: Not Started**

---

## Phase 6 Context and Constraints

- Replace the Phase 1 `ReportScreen` placeholder with a working PDF report generator.
- All entry data read through `DiaryService` (Phase 2). No direct Hive calls in UI or service code.
- Symptom names resolved through `SymptomsNotifier` (Phase 4). No direct Hive calls for symptom resolution.
- PDF generated in memory (`Uint8List`) and shared via Android share sheet. No persistent plaintext file written.
- `printing` package writes a transient file to app-private cache for the Android share intent — this is acceptable (private to app, not persisted, not user-accessible storage).
- Android-first. iOS not in scope for v1.
- OQ1 (`pdf + printing` vs alternative) and OQ2 (layout: section per entry vs table) are open and non-blocking — defaults applied. Note OQ1 and OQ2 status in return handoff.
- Rapid delivery is a product priority.

---

## Phase 6 Implementation Scope

Follow `Plan_Phase6_Printable_Reports.md` stages in order. All decisions are finalised.

### Stage 1 — Add Dependencies

Add to `pubspec.yaml` and run `flutter pub get`:

```yaml
dependencies:
  pdf: ^3.10.0
  printing: ^5.12.0
```

Verify `flutter analyze` remains clean after package resolution.

### Stage 2 — Report Service

Create `lib/features/reports/services/report_service.dart`:

- `ReportService` class with a single public method:

  ```dart
  Future<Uint8List> generatePdf(
    List<DailyEntry> entries,
    List<Symptom> symptoms,
    DateTimeRange range,
  )
  ```

- **Method is async** — calls `await doc.save()` internally (`pdf` package `Document.save()` returns `Future<Uint8List>`). No other I/O; no Hive dependency; no Riverpod dependency.

  **Implementation note N1 (LOW — stale assumption):** Plan Assumption 3 still reads “synchronous” — this is a stale carry-over from before the M1 fix. Stage 2 is the authoritative specification. Implement `generatePdf()` as async per Stage 2. Disregard Assumption 3.

- PDF structure:
  - **Header**: “Symptom Diary Report”, date range formatted as “DD MMM YYYY – DD MMM YYYY”.
  - **Entry sections** (one per entry, sorted by date ascending): date line, pain level (“Pain: N/10”), symptoms (“Symptoms: name1, name2, …”; if none: “Symptoms: None”), notes paragraph (omit line if empty).
  - **Footer**: “Generated: [generation date]”.
- `reportServiceProvider = Provider<ReportService>((ref) => ReportService())` in `lib/features/reports/providers/report_provider.dart`.

  **Implementation note N2 (LOW — stale progress sheet):** Progress sheet row for Stage 2 CC reads “produces non-empty `Uint8List`” — should read `Future<Uint8List>`. Stage 2’s own CC (“returns a non-empty `Future<Uint8List>`”) is correct. Follow Stage 2 CC.

### Stage 3 — Report Screen

Replace `ReportScreen` Phase 1 placeholder at `lib/features/reports/screens/report_screen.dart`:

- `ReportScreen` is a `ConsumerStatefulWidget` with local `_isLoading`, `_dateRange`, and `_errorMessage` state fields.
- Date range row: `OutlinedButton` labelled with current range (or “Select date range”). Taps open `showDateRangePicker`. `lastDate`: today normalised. `firstDate`: `DateTime(2000)`.
- Generate button: disabled when `_dateRange` is null or `_isLoading` is true. Shows `CircularProgressIndicator` during generation.
- Error message shown below button if `_errorMessage` is non-null.
- Generate flow:

  ```dart
  Future<void> _generate(WidgetRef ref) async {
    setState(() { _isLoading = true; _errorMessage = null; });
    try {
      final service = ref.read(diaryServiceProvider);
      final allEntries = await service.getAllEntries();
      final filtered = allEntries
          .where((e) => !e.date.isBefore(_dateRange!.start)
                     && !e.date.isAfter(_dateRange!.end))
          .toList()
        ..sort((a, b) => a.date.compareTo(b.date));
      if (filtered.isEmpty) {
        setState(() { _errorMessage = 'No entries found for this date range.'; });
        return;
      }
      final allSymptoms = await ref.read(symptomsProvider.future);
      final bytes = await ref.read(reportServiceProvider)
          .generatePdf(filtered, allSymptoms, _dateRange!);
      await Printing.sharePdf(
        bytes: bytes,
        filename: 'symptom_report_${_dateRange!.start.toIso8601String().substring(0, 10)}.pdf',
      );
    } catch (e) {
      setState(() { _errorMessage = 'Report generation failed. Please try again.'; });
    } finally {
      setState(() { _isLoading = false; });
    }
  }
  ```

### Stage 4 — Tests

- `ReportService.generatePdf()`: single entry → non-empty bytes; multiple entries → bytes larger than single entry; empty entries list → still returns non-empty PDF (empty-state handled in screen, not service).
- `ReportService.generatePdf()` symptom resolution: entry with known `symptomIds` — assert no throw + non-empty bytes. **Do NOT assert symptom name literals in raw `Uint8List` bytes** — PDF content streams encode text via font tables; string presence in raw bytes is unreliable across `pdf` package versions.
- `ReportService.generatePdf()`: entry with no symptoms → no crash; entry with no notes → no crash.
- Widget test `ReportScreen`: Generate button disabled with no date range; shows “No entries found” message when `diaryServiceProvider` returns empty list for range.
- All provider tests use `ProviderScope(overrides: [...])` with mock services.

### Stage 5 — Validation

1. `dart format lib/ test/`
2. `flutter analyze` — zero issues
3. `flutter test` — all tests pass
4. `flutter build apk --debug` — succeeds
5. Manual smoke on emulator: Reports tab → select date range that includes entries → Generate → loading indicator → Android share sheet opens → open PDF → verify header, entry sections, footer readable. Then: select range with no entries → verify “No entries found” shown; share sheet not triggered.

---

## Phase 6 Acceptance Criteria (summary — full list in plan AC 1–19)

All 19 AC items in `Plan_Phase6_Printable_Reports.md` must pass. Key items: date range picker works (AC1–3); loading indicator shown during generation (AC4); PDF in memory, no persistent file (AC5); PDF includes header, entry sections, footer (AC6–8); share sheet opens on success (AC9); empty-range message without share (AC10); error message on failure (AC11); no API 32+ (AC12); `ReportService.generatePdf()` is async returning `Future<Uint8List>`, no Hive, no Riverpod (AC13); unit tests cover populated/no-symptoms/no-notes (AC14); widget tests cover disabled state and empty-result (AC15); format/analyze/test/build (AC16–18); smoke (AC19).

---

## Phase 6 Return Handoff Requirements

When all 5 stages complete and all 19 AC pass, update `Documentation/Handoffs/df-coder_to_df-qa.md` with:

- Stage completion status for all 5 stages
- Validation output summary (format, analyze, test, build, smoke)
- Emulator/device used for smoke test
- Confirmation that N1 (async generatePdf) and N2 (progress sheet CC note) implementation notes were noted
- OQ1 and OQ2 status (note defaults applied if unresolved — state explicitly)
- Any deviations from the plan (if none, state explicitly)

---

## Phase 7 Context and Constraints

- Phase 7 introduces no new features. All work is UX review, polish, hardening, and release build preparation.
- Implement only after Phases 1–6 are complete and QA-approved.
- All UX fixes must be minimum-change. If a UX issue implies feature work, defer to a post-v1 issue.
- ProGuard/R8 minification is disabled for v1 (`minifyEnabled false`, `shrinkResources false`).
- Release signing: OQ1 open — Stage 4 covers both paths (generate new keystore vs. use existing). Debug-signed APK is acceptable for Phase 7 plan validation but not for publication.

---

## Phase 7 Implementation Scope

Follow `Plan_Phase7_Polish_Release.md` stages in order: 1 → 2 → 3 → 4 → 5 → 6. All decisions are finalised — do not deviate without raising a new handoff.

### Stage 1 — UX Review Pass

Systematically navigate every screen and catalogue issues. No fixes — record only. Review checklist: navigation reachability and pop behaviour, lock screen `PopScope`, empty states, error states, loading states, date picker restrictions (≤ today), presets empty state, symptom picker overflow.

**Also explicitly check:** `EntryDetailScreen` null guard — if `diaryProvider(date)` returns null for the selected date, the screen must show "No entry for this date" + back button (Phase 5 Stage 3 requirement). This is not a list-empty state; it is a specific null-entry path.

### Stage 2 — UX Fixes

Apply all Stage 1 issues (minimum scope). Fix patterns:
- Navigation dead ends: route registration in `app.dart`.
- Missing empty states: `if (list.isEmpty) const _EmptyState()` inline private widget.
- Missing error states: `AsyncValue.when(error: ...)` user-facing message widget.
- Missing loading indicators: centred `const CircularProgressIndicator()`.
- `PopScope(canPop: false, onPopInvokedWithResult: (_, __) {})` on `LockScreen` only.

After all fixes: `dart format lib/ test/`, `flutter analyze` (zero issues), `flutter test` (all pass).

### Stage 3 — Keyboard and Scroll Hardening

Apply keyboard/scroll checklist to all text-input screens (`LogEntryScreen`, `AddSymptomScreen`, preset save dialog, `SettingsScreen`). Critical per-field distinction:

- **Single-line last fields** (`AddSymptomScreen` name, preset save dialog name): `textInputAction: TextInputAction.done`.
- **Multiline notes field** (`LogEntryScreen`): leave at Flutter default `TextInputAction.newline` (or set explicitly) — do **NOT** use `TextInputAction.done`. Keyboard dismissed by outer `GestureDetector(onTap: () => FocusScope.of(context).unfocus())`.
- **Intermediate fields**: `textInputAction: TextInputAction.next`; `onSubmitted: (_) => FocusScope.of(context).nextFocus()`.
- All screens: `Scaffold(resizeToAvoidBottomInset: true)` confirmed; form body in `SingleChildScrollView` if not already present.

After changes: `dart format lib/ test/`, `flutter analyze` (zero issues), `flutter test` (all pass).

### Stage 4 — Release Build

1. In `android/app/build.gradle` release build type: `minifyEnabled false`, `shrinkResources false`.
2. Create `android/key.properties` (not committed): `storePassword`, `keyPassword`, `keyAlias`, `storeFile` (absolute path to `release.jks` stored **outside the repository**).
3. Add `key.properties` and `*.jks` to `android/.gitignore`.
4. Add groovy reader block at top of `android/app/build.gradle`:
   ```groovy
   def keystoreProperties = new Properties()
   def keystorePropertiesFile = rootProject.file('key.properties')
   if (keystorePropertiesFile.exists()) {
       keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
   }
   ```
5. Add `signingConfigs.release` block reading from `keystoreProperties` (no hardcoded values). Update `buildTypes.release.signingConfig` to `signingConfigs.release`. Define `signingConfigs` block **before** `buildTypes` in the gradle file.
6. If OQ1 unresolved (no keystore available), retain `signingConfig signingConfigs.debug` as placeholder and note this explicitly in the return handoff.
7. Run `flutter build apk --release`. Verify APK at `build/app/outputs/flutter-apk/app-release.apk`.

### Stage 5 — Documentation Updates

1. Create `Documentation/Records/Implementation-Summaries/v1.0.0.md` (create the `Implementation-Summaries/` directory if absent): app overview paragraph, Phases 1–6 completion table, key technical decisions (Hive + AES-256, `flutter_secure_storage`, `local_auth`, Riverpod, Navigator 1.0 named routes, `table_calendar ^3.1.0`, `pdf ^3.10.0` + `printing ^5.12.0`), known limitations (iOS not v1, ProGuard disabled, any open OQs).
2. Update `README.md` with: prerequisites (Flutter SDK exact version, Android SDK), build commands (`flutter pub get`, `flutter build apk --debug`, `flutter build apk --release`), test command (`flutter test`), code generation command (`dart run build_runner build --delete-conflicting-outputs`).

### Stage 6 — Final Validation

1. `dart format lib/ test/` — no changes.
2. `flutter analyze` — zero issues.
3. `flutter test` — all tests pass.
4. `flutter build apk --debug` — succeeds.
5. `flutter build apk --release` — succeeds.
6. Manual end-to-end smoke (7 flows): cold launch → auth → home; log entry today; log entry past date; history pull-to-refresh + filter + clear; report generation (share sheet, PDF readable); settings idle timeout change; idle timeout mid-session fires lock screen → re-auth → return.

---

## Phase 7 Implementation Notes

**L1 — LOW:** Stage 1 UX checklist does not include an explicit row for `EntryDetailScreen` null guard. Phase 5 Stage 3 specifies: if `diaryProvider(date)` returns null (date has no logged entry), the screen must show "No entry for this date" text and a back button — not a blank screen or null render. Verify this is correctly implemented during the Stage 1 UX review pass.

---

## Phase 7 Acceptance Criteria (summary — full list in plan AC 1–14)

All 14 AC items in `Plan_Phase7_Polish_Release.md` must pass. Key items: all screens reachable (AC1); lock screen `PopScope(canPop: false)` (AC2); empty state on all list screens (AC3); async screens handle loading, data, and error branches (AC4); notes field scrolls without obscuring (AC5); `TextInputAction.done` for single-line last fields, `TextInputAction.newline` for multiline notes field (AC6); `flutter analyze` zero issues (AC7); `dart format` clean (AC8); `flutter test` all pass (AC9); debug build succeeds (AC10); release build with ProGuard disabled and `key.properties`/`*.jks` gitignored (AC11); `v1.0.0.md` exists and complete (AC12); `README.md` updated (AC13); end-to-end smoke 7 flows (AC14).

---

## Phase 7 Return Handoff Requirements

When all 6 stages complete and all 14 AC pass, update `Documentation/Handoffs/df-coder_to_df-qa.md` with:

- Stage completion status for all 6 stages
- Validation output summary (format, analyze, test, debug build, release build, smoke)
- Emulator/device used for smoke test
- OQ1 status (signing key present or absent — state explicitly; note if debug-signed only)
- OQ2 status (smoke test device — emulator or real device)
- Confirmation that L1 (`EntryDetailScreen` null guard) was checked during Stage 1
- Any deviations from the plan (if none, state explicitly)

---

## Handoff Target

df-qa — via `Documentation/Handoffs/df-coder_to_df-qa.md`
