# Plan: Phase 1 — Project Scaffold

**Created:** 2026-06-02
**Status:** Approved — Phase 1 Ready for Implementation
**Parent Plan:** `Documentation/Plans/Plan_Initial_App_Delivery.md`
**Active Phase:** Phase 1 — Project Scaffold

---

## Objective

Create a buildable, installable Flutter project with the correct directory structure, navigation shell, theming, Android SDK configuration, and Riverpod scaffolding. No feature logic is implemented in this phase. The output is a skeleton app that compiles, passes analysis, and runs on an Android device or emulator.

---

## Constraints

- No feature logic. Screens are placeholder widgets only.
- No authentication logic, no storage, no data models.
- Rapid delivery priority: keep dependency count minimal.
- All workflow artifacts must remain inside this repository under `Documentation/`.
- Android SDK: minSdkVersion 23, targetSdkVersion 31, compileSdkVersion 31.

---

## Assumptions

1. Flutter SDK is installed and `flutter doctor` is clean on the dev machine.
2. Android SDK and build tools compatible with API 31 are available.
3. App package identifier: `org.zephon.symptom_diary`.
4. App display name: `Symptom Diary`.
5. Flutter project name (directory and `pubspec.yaml` name field): `symptom_diary`.
6. Bottom navigation bar is the selected navigation pattern (see rationale in Stage 3).
7. Material 3 with a seed-based color scheme is sufficient for v1 theming. No custom design system is required.
8. Navigator 1.0 (push/pop with named routes) is used for Phase 1. A routing package upgrade can be addressed during a later phase if needed.
9. The only Riverpod provider created in Phase 1 is a simple `navigationIndexProvider` (`StateProvider<int>`) to manage the selected bottom nav tab. All feature providers are deferred to their respective phases.
10. `flutter_riverpod` version `^2.6.0` is used (latest stable at time of writing; adjust to latest stable at time of implementation).

---

## Detailed Stage Plan

### Stage 1 — Flutter Project Creation

Run `flutter create` with the correct parameters and confirm the output directory structure is generated correctly.

**Before running `flutter create` — preserve the existing README:**

The repository already has a `README.md` at root. `flutter create` will overwrite it silently. Before running the command, copy the existing file:

```sh
copy README.md README.md.bak
```

After `flutter create` completes, delete the generated Flutter README and restore the original:

```sh
del README.md
rename README.md.bak README.md
```

**Command:**

```sh
flutter create --org org.zephon --project-name symptom_diary .
```

> This targets the repository root (`.`). Confirm the generated `pubspec.yaml` `name` field reads `symptom_diary` and `applicationId` in `android/app/build.gradle` reads `org.zephon.symptom_diary`.

**Deliverables:**

- Flutter project generated at repository root.
- Default `lib/main.dart` and counter app present (to be replaced in Stage 3).
- `pubspec.yaml` name: `symptom_diary`.
- Android application ID: `org.zephon.symptom_diary`.
- Existing `README.md` preserved; default Flutter README replaced.

---

### Stage 2 — Directory Structure

Establish the feature-based directory layout. Create all directories and stub files. No logic is written; files contain only a minimal class or placeholder comment.

**Target structure:**

```text
lib/
  main.dart
  app.dart
  core/
    theme/
      app_theme.dart
  features/
    auth/
      screens/
        lock_screen.dart
    diary/
      screens/
        home_screen.dart
        log_entry_screen.dart
    history/
      screens/
        history_screen.dart
    reports/
      screens/
        report_screen.dart
    settings/
      screens/
        settings_screen.dart
    shared/
      providers/
        navigation_provider.dart
```

The `shared/providers/` directory contains cross-feature providers. `navigation_provider.dart` is a stub file in Stage 2; its content is defined in Stage 5. Directory names match CODING_STANDARDS snake_case convention.

**Deliverables:**

- All directories created, including `lib/features/shared/providers/`.
- Each `_screen.dart` file contains a minimal stateless placeholder widget that displays the screen name as a `Text` widget. No logic.
- `lib/features/shared/providers/navigation_provider.dart` created as a stub (content added in Stage 5).
- `app.dart` exists (content added in Stage 3).
- `core/theme/app_theme.dart` exists (content added in Stage 4).

---

### Stage 3 — Navigation Shell

Replace the default counter app with a navigation shell that wires the bottom navigation bar to the placeholder screens.

**Navigation decision — Bottom Navigation Bar:**

Rationale: The app has exactly 4 top-level destinations (Home, History, Reports, Settings). Bottom navigation is the Material Design standard for 3–5 top-level Android destinations, is thumb-friendly, and is widely expected by Android users. A navigation drawer adds unnecessary complexity for 4 items.

The lock screen is a full-screen route separate from the shell. It is implemented as a placeholder in Phase 1 (navigated to from `main.dart` directly via `Navigator.pushNamed`). The auth integration is Phase 3 work.

**Navigation tabs:**

| Index | Label | Icon | Screen |
| --- | --- | --- | --- |
| 0 | Home | `Icons.home_outlined` / `Icons.home` | `HomeScreen` |
| 1 | History | `Icons.calendar_month_outlined` / `Icons.calendar_month` | `HistoryScreen` |
| 2 | Reports | `Icons.description_outlined` / `Icons.description` | `ReportScreen` |
| 3 | Settings | `Icons.settings_outlined` / `Icons.settings` | `SettingsScreen` |

**Routing:**

- `main.dart`: wraps `app.dart` content with `ProviderScope` (Stage 5). Entry screen is the nav shell.
- `app.dart`: contains `MaterialApp` with `AppTheme.lightTheme` applied (Stage 4), and `initialRoute` pointing to `/`.
- Named routes defined: `/` → `MainShell`, `/lock` → `LockScreen`.
- `MainShell` is a `ConsumerStatefulWidget` that manages `navigationIndexProvider` and renders an `IndexedStack` of the 4 tab screens.

**Deliverables:**

- `main.dart` boots the app (ProviderScope wrap added in Stage 5; use a temporary bare `runApp` in Stage 3 if needed until Stage 5 is complete).
- `app.dart` contains `MaterialApp` with named routes.
- `MainShell` widget renders the bottom navigation bar and an `IndexedStack` of the 4 placeholder screens.
- Navigation between tabs works via the bottom nav bar.
- `/lock` route renders the `LockScreen` placeholder.
- Hot reload works without errors.

---

### Stage 4 — App Theming

Define the application theme in `core/theme/app_theme.dart` and apply it in `app.dart`.

**Theme decisions:**

- Material 3 (`useMaterial3: true`).
- Seed color: `Color(0xFF1565C0)` (Material Blue 800) — a calm, medical-context appropriate blue. Can be changed before Phase 7 polish without structural rework.
- `ColorScheme.fromSeed(seedColor: ..., brightness: Brightness.light)` for light theme.
- `ColorScheme.fromSeed(seedColor: ..., brightness: Brightness.dark)` for dark theme.
- `ThemeMode.system` in `MaterialApp` so the OS preference is respected automatically.
- No custom typography in Phase 1; use M3 defaults (`TextTheme` unset).

**Deliverables:**

- `app_theme.dart` exposes `AppTheme.lightTheme` and `AppTheme.darkTheme` as static `ThemeData` getters.
- `MaterialApp` in `app.dart` sets `theme`, `darkTheme`, and `themeMode: ThemeMode.system`.
- App renders visually consistent colors and responds to OS dark/light mode toggle.

---

### Stage 5 — Riverpod Scaffold

Add `flutter_riverpod` to `pubspec.yaml`, wrap the app with `ProviderScope`, and define the single Phase 1 provider.

**`pubspec.yaml` additions:**

```yaml
dependencies:
  flutter_riverpod: ^2.6.0
```

**Providers defined in Phase 1:**

| Provider | Type | Location | Purpose |
| --- | --- | --- | --- |
| `navigationIndexProvider` | `StateProvider<int>` | `lib/features/shared/providers/navigation_provider.dart` | Tracks selected bottom nav tab index |

> **Note — approved exception:** CODING_STANDARDS prefers `NotifierProvider` for synchronous state. `StateProvider<int>` is used here as an approved exception for trivial single-value scaffold state in Phase 1. If the navigation provider grows in complexity in a later phase, it should be migrated to `NotifierProvider`.

All other feature providers are deferred. Empty provider directories are created as stubs only where the CODING_STANDARDS co-location pattern requires a `providers/` subdirectory.

**Deliverables:**

- `flutter_riverpod` in `pubspec.yaml`, `flutter pub get` clean.
- `ProviderScope` wraps `MaterialApp` in `main.dart`.
- `navigationIndexProvider` defined and consumed by `MainShell`.
- No other providers created.
- `lib/features/shared/providers/navigation_provider.dart` created.

---

### Stage 6 — Android SDK Configuration

Update `android/app/build.gradle` to match the platform targets in CODING_STANDARDS.

**Required values:**

```groovy
compileSdkVersion 31
defaultConfig {
    minSdkVersion 23
    targetSdkVersion 31
}
```

Confirm `applicationId` is `org.zephon.symptom_diary` (set by `flutter create` in Stage 1; verify only, do not change unless wrong).

**Deliverables:**

- `compileSdkVersion 31`, `minSdkVersion 23`, `targetSdkVersion 31` confirmed in `android/app/build.gradle`.
- `applicationId` confirmed as `org.zephon.symptom_diary`.

---

### Stage 7 — Validation

Run all required checks before marking Phase 1 complete.

**Checks:**

| Check | Command | Pass Criteria |
| --- | --- | --- |
| Formatter | `dart format lib/` | No unformatted files reported |
| Analyzer | `flutter analyze` | No issues (warnings or errors) |
| Debug build | `flutter build apk --debug` | Build completes without errors |
| Manual smoke | Run on emulator or device | App launches, bottom nav works, all 4 tabs navigate, no runtime errors in console |

**Deliverables:**

- `dart format` clean.
- `flutter analyze` clean.
- Debug APK builds without errors.
- Manual smoke test on Android emulator documented in handoff (pass/fail per check, device/emulator used).

---

## Progress Sheet

| Stage | Owner | Status | Completion Criteria | Notes |
| --- | --- | --- | --- | --- |
| Stage 1 — Flutter Project Creation | df-coder | Not Started | Project created; pubspec name = `symptom_diary`; applicationId = `org.zephon.symptom_diary`; existing `README.md` preserved | Preserve `README.md` before running `flutter create` (see Stage 1 instructions) |
| Stage 2 — Directory Structure | df-coder | Not Started | All directories and stub files present including `shared/providers/`; snake_case names throughout | `navigation_provider.dart` stub created here; content added in Stage 5 |
| Stage 3 — Navigation Shell | df-coder | Not Started | Bottom nav working; 4 tabs navigate to placeholder screens; lock route defined | `ConsumerStatefulWidget` for shell |
| Stage 4 — App Theming | df-coder | Not Started | `AppTheme` applied; M3 seed scheme; dark/light responds to OS | Seed color: `Color(0xFF1565C0)` |
| Stage 5 — Riverpod Scaffold | df-coder | Not Started | `ProviderScope` wraps app; `navigationIndexProvider` defined and consumed | No other providers in Phase 1 |
| Stage 6 — Android SDK Config | df-coder | Not Started | `compileSdkVersion 31`, `minSdkVersion 23`, `targetSdkVersion 31` confirmed in build.gradle | Verify only unless wrong |
| Stage 7 — Validation | df-coder | Not Started | Format clean; analyze clean; debug APK builds; manual smoke pass | Document emulator/device used |

---

## Acceptance Criteria

1. `pubspec.yaml` `name` field is `symptom_diary` and `applicationId` in `android/app/build.gradle` is `org.zephon.symptom_diary`.
2. Directory structure matches Stage 2 layout exactly. All file and directory names are snake_case.
3. App launches on Android without a runtime error.
4. Bottom navigation bar has exactly 4 tabs: Home, History, Reports, Settings.
5. Each tab navigates to a named placeholder screen displaying the screen's name.
6. The `/lock` route renders the `LockScreen` placeholder.
7. `ProviderScope` wraps `MaterialApp` in `main.dart`.
8. `navigationIndexProvider` (`StateProvider<int>`) is the only provider defined; it is consumed by the navigation shell.
9. `AppTheme.lightTheme` and `AppTheme.darkTheme` are applied; M3 is enabled; theme responds to OS dark/light mode.
10. Android SDK values: `minSdkVersion 23`, `targetSdkVersion 31`, `compileSdkVersion 31`.
11. `dart format lib/` reports no changes needed.
12. `flutter analyze` reports zero issues.
13. `flutter build apk --debug` completes without errors.

---

## Validation Approach

- `dart format lib/` and `flutter analyze` must both pass before Stage 7 is marked complete.
- Debug APK build (`flutter build apk --debug`) is the final gate.
- Manual smoke test on Android emulator (API 31 recommended) to verify runtime behavior.
- df-coder must document which emulator/device was used and confirm each validation check in the handoff to df-qa.
- Any failing checks must be resolved before routing to df-qa.

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| Flutter SDK version too old to target API 31 cleanly | Low | Medium | Check `flutter --version` and `flutter doctor`; upgrade if needed |
| `flutter create` with `--org` produces wrong `applicationId` | Low | Low | Verify in `android/app/build.gradle` in Stage 1 before proceeding |
| Android toolchain not configured (missing SDK or NDK) | Medium | High | Run `flutter doctor` and resolve before Stage 1 |
| `flutter_riverpod ^2.6.0` has a breaking change at implementation time | Low | Low | Adjust version pin to latest stable; verify `flutter pub get` clean |
| `IndexedStack` causes memory concerns with future complex screens | Low | Low | Acceptable for scaffold; review during Phase 4 if screens become heavy |

---

## Open Questions

| ID | Question | Status |
| --- | --- | --- |
| OQ1 | Should the Flutter project live at the repository root or in a subdirectory (e.g., `app/`)? | **Closed — confirmed repo root.** QA inspected repo root: only `.git/`, `.github/`, `Documentation/`, `README.md` present. No structural conflict. |
| OQ2 | Is a specific Flutter SDK version pinned for this project? | **Closed — no pin required.** Latest stable Flutter SDK is acceptable. |

---

## Deferred to Later Phases

- Authentication logic: Phase 3.
- All data models and storage: Phase 2.
- All feature providers other than `navigationIndexProvider`: their respective phases.
- Routing package upgrade (e.g., go_router): deferred unless a blocker is encountered in Phase 4 or later.
- Log entry screen navigation (navigating *to* the log screen from home): Phase 4.

---

## Handoff Target

df-qa
