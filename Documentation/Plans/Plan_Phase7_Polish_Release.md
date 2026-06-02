# Plan: Phase 7 — Polish and Release Preparation

**Created:** 2026-06-02  
**Status:** Approved — Ready for Implementation (after Phase 6)  
**Parent Plan:** `Documentation/Plans/Plan_Initial_App_Delivery.md`  
**Prerequisite:** Phase 6 complete and approved by df-qa.  
**Active Phase:** Phase 7 — Polish and Release Preparation

---

## Objective

Review the completed app end-to-end for UX issues, harden keyboard and scroll behaviour, produce a passing release-mode Android APK, verify the code analysis and test suite are clean, and update documentation for handoff and future maintenance. No new features.

---

## Constraints

- No new features or scope expansion in this phase. If the UX review surfaces a feature gap rather than a bug, it is deferred to a post-v1 issue.
- UX fixes must not break existing unit or widget tests.
- Rapid delivery and low complexity remain product priorities.
- All workflow artifacts must remain inside this repository under `Documentation/`.
- Android-first. iOS not in scope for v1.

---

## Assumptions

1. Phases 1–6 are complete and QA-approved before Phase 7 begins.
2. `flutter analyze`, `flutter test`, and `dart format` are all passing at the end of Phase 6.
3. A release signing keystore may or may not exist at Phase 7 start — Stage 4 documents both paths; coordinator/owner must supply or generate the key before a publishable APK is possible.
4. ProGuard/R8 minification is disabled for v1 release builds (see Stage 4). This avoids runtime class-stripping issues with Hive adapters and `flutter_secure_storage`. Enabling minification is deferred to a post-v1 hardening cycle.
5. Emulator smoke test is acceptable for Phase 7 plan validation. Real-device verification of biometric auth is a recommended post-release step, not a plan gate.
6. `Documentation/Records/Implementation-Summaries/` may not exist yet — df-coder creates it if absent.

---

## Open Questions

| ID  | Question | Status |
| --- | -------- | ------ |
| OQ1 | Release signing keystore: does one exist, or is first-time generation required? | Open — coordinator/owner input before Stage 4; Stage 4 covers both paths. |
| OQ2 | Phase 7 smoke test device: emulator only, or specific real device required for sign-off? | Open — default: emulator acceptable. |

---

## Directory Structure (additions)

```
Documentation/
  Records/
    Implementation-Summaries/
      v1.0.0.md                       # Created in Stage 5
```

No new `lib/` files. Any UX fix modifies existing screen files only.

---

## Stage Plan

### Stage 1 — UX Review Pass

Systematically navigate every screen and catalogue issues. No fixes in this stage — record only.

Review checklist (each item must pass or be logged with screen name, severity, and fix description):

| Area | Check |
| --- | --- |
| Navigation | Every screen reachable from the bottom navigation shell |
| Navigation | Back/pop on every screen returns to the expected parent without a double-pop or dead end |
| Lock screen | `PopScope(canPop: false)` in place; back button does not dismiss lock screen to home |
| Empty states | Every list screen shows a message widget when its list is empty (not a blank or null render) |
| Error states | Every async screen handles the error branch with a user-facing message (not a raw exception string) |
| Loading states | Every async screen shows a `CircularProgressIndicator` (or equivalent) while loading |
| Date pickers | Log entry date picker restricted to ≤ today; report and filter date pickers restricted to lastDate = today |
| Presets screen | Empty preset list shows appropriate empty state message |
| Symptom picker | Combined predefined + custom list renders without overflow or layout error |

**Owner:** df-coder  
**Completion Criteria:** Every checklist item reviewed. All issues logged with screen name, severity (Low/Medium/High), and fix description. No fix work performed yet.

---

### Stage 2 — UX Fixes

Apply all issues logged in Stage 1. Scope each fix to the minimum change:

- **Navigation dead ends:** add or correct route registrations in `app.dart`.
- **Missing empty states:** add `if (list.isEmpty) const _EmptyState()` guards in the affected screen. Use a concise inline private widget.
- **Missing error states:** ensure `AsyncValue.when(data: ..., loading: ..., error: ...)` has a user-facing error widget in the `error` branch (not `throw` or silent swallow).
- **Missing loading indicators:** add `const CircularProgressIndicator()` (centred) in the `loading` branch.
- **PopScope issues:** `PopScope(canPop: false, onPopInvokedWithResult: (_, __) {})` on `LockScreen` only. All other screens should allow back navigation.

If a fix requires touching a provider, keep the change minimal — update `state` directly; do not restructure a notifier.

After all fixes:

1. `dart format lib/ test/`
2. `flutter analyze` — zero issues
3. `flutter test` — all tests pass

**Owner:** df-coder  
**Completion Criteria:** All Stage 1 issues resolved. `flutter analyze` clean. `flutter test` passing. No new features introduced.

---

### Stage 3 — Keyboard and Scroll Hardening

Verify and fix keyboard/scroll behaviour on all screens that contain text input:

- `LogEntryScreen` — notes field (multiline), date display
- `AddSymptomScreen` — name field
- Preset save dialog — name field
- `SettingsScreen` — any text input present

Checklist per screen with text input:

| Check | Action |
| --- | --- |
| `Scaffold(resizeToAvoidBottomInset: true)` | Verify this is set (it is the Flutter default — confirm explicitly) |
| Form fields not obscured when keyboard opens | Wrap form body in `SingleChildScrollView` if not already present |
| Single-line last fields (`AddSymptomScreen` name, preset save dialog name) use `TextInputAction.done` | Set `textInputAction: TextInputAction.done` |
| Multiline notes field (`LogEntryScreen`) — do **NOT** use `TextInputAction.done` | Leave at Flutter default `TextInputAction.newline` (or set explicitly); keyboard dismissed by outer `GestureDetector` tap-outside |
| Intermediate fields use `TextInputAction.next` | Set `textInputAction: TextInputAction.next`; `onSubmitted: (_) => FocusScope.of(context).nextFocus()` |
| Tapping outside a text field dismisses keyboard | Wrap screen body in `GestureDetector(onTap: () => FocusScope.of(context).unfocus(), child: ...)` |

After changes:

1. `dart format lib/ test/`
2. `flutter analyze` — zero issues
3. `flutter test` — all tests pass

**Owner:** df-coder  
**Completion Criteria:** All text-input screens scroll correctly with keyboard open; no field obscured. `TextInputAction.done` set on all single-line last fields; multiline notes field (`LogEntryScreen`) uses `TextInputAction.newline` (not `done`). Keyboard dismiss via tap-outside works on all screens. Analyze and tests pass.

---

### Stage 4 — Release Build

**Pre-step — Disable ProGuard/R8 (required):**

In `android/app/build.gradle`, set the release build type:

```groovy
buildTypes {
    release {
        minifyEnabled false
        shrinkResources false
        signingConfig signingConfigs.debug   // temporary — see OQ1
    }
}
```

This prevents Hive-generated adapter classes and `flutter_secure_storage` from being stripped at runtime. Enabling minification is deferred to a post-v1 hardening cycle.

**Signing (OQ1):**

Two paths — choose based on OQ1 resolution:

**(A) No keystore exists (first-time setup):** Generate with:

```sh
keytool -genkey -v -keystore release.jks -keyalg RSA -keysize 2048 -validity 10000 -alias release
```

Store `release.jks` in a **secure location outside the repository** (never commit it to source control).

**(B) Keystore already exists:** Retrieve from secure credential store.

**Credential injection (both paths — required):**

Never hardcode `storePassword`, `keyPassword`, `storeFile`, or `keyAlias` directly in `android/app/build.gradle`. Use the `key.properties` file pattern:

1. Create `android/key.properties` (not committed):

   ```properties
   storePassword=<password>
   keyPassword=<password>
   keyAlias=release
   storeFile=<absolute-path-to-release.jks>
   ```

2. Add both entries to `android/.gitignore`:

   ```
   key.properties
   *.jks
   ```

3. Read `key.properties` at the top of `android/app/build.gradle`:

   ```groovy
   def keystoreProperties = new Properties()
   def keystorePropertiesFile = rootProject.file('key.properties')
   if (keystorePropertiesFile.exists()) {
       keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
   }
   ```

4. Configure `signingConfigs.release` to read from `keystoreProperties`:

   ```groovy
   signingConfigs {
       release {
           storeFile keystoreProperties['storeFile'] ? file(keystoreProperties['storeFile']) : null
           storePassword keystoreProperties['storePassword']
           keyAlias keystoreProperties['keyAlias']
           keyPassword keystoreProperties['keyPassword']
       }
   }
   ```

5. Update `buildTypes.release.signingConfig` to `signingConfigs.release`.

The `if (keystorePropertiesFile.exists())` guard allows graceful fallback to `signingConfigs.debug` when the properties file is absent (e.g. CI without secrets configured).

If OQ1 is unresolved (no key available by Stage 4), retain `signingConfig signingConfigs.debug` as a verification step only. Note this explicitly in the return handoff. A debug-signed APK is not suitable for publication.

**Build:**

```sh
flutter build apk --release
```

Verify APK produced at `build/app/outputs/flutter-apk/app-release.apk`.

**Owner:** df-coder  
**Completion Criteria:** `flutter build apk --release` succeeds and produces `app-release.apk`. ProGuard/R8 disabled. `key.properties` and `*.jks` in `android/.gitignore`; no credentials hardcoded in `build.gradle`. OQ1 status (signing key present or absent) noted explicitly in return handoff.

---

### Stage 5 — Documentation Updates

1. Create `Documentation/Records/Implementation-Summaries/v1.0.0.md` containing:
   - **App overview** (one paragraph): purpose, platform, key features.
   - **Phase completion table** (Phases 1–6, completion dates, QA-approved status).
   - **Key technical decisions**: Hive + AES-256 encryption, `flutter_secure_storage` for key, `local_auth` for authentication, Riverpod for state management, Navigator 1.0 named routes, `table_calendar ^3.1.0`, `pdf ^3.10.0` + `printing ^5.12.0`.
   - **Known limitations**: iOS not supported (v1); ProGuard/R8 minification disabled; any OQs from Phases 4–6 still open.

2. Update `README.md` with:
   - **Prerequisites**: Flutter SDK version used (record the exact version), Android SDK, required environment setup.
   - **Build commands**: `flutter pub get`, `flutter build apk --debug`, `flutter build apk --release`.
   - **Test command**: `flutter test`.
   - **Code generation** (if needed post-clean): `dart run build_runner build --delete-conflicting-outputs`.

**Owner:** df-coder  
**Completion Criteria:** `v1.0.0.md` exists and is complete. `README.md` updated with correct commands a new developer can follow to build from source.

---

### Stage 6 — Final Validation

All five checks must pass:

| Check | Command | Required Outcome |
| --- | --- | --- |
| Format | `dart format lib/ test/` | No changes |
| Analyzer | `flutter analyze` | Zero issues |
| Tests | `flutter test` | All tests pass |
| Debug build | `flutter build apk --debug` | Succeeds |
| Release build | `flutter build apk --release` | Succeeds |

Manual end-to-end smoke (full app flow):

1. Cold launch → lock screen → authenticate → home.
2. Log entry (today): pain level + symptoms + notes → save → home reflects entry.
3. Log entry (past date): backdated entry → save → history calendar shows entry.
4. History: pull-to-refresh → apply filter (date range + symptom + pain threshold) → clear filters → empty state shown when no match.
5. Reports: select date range → Generate → loading indicator → Android share sheet opens → open PDF → header, entry sections, and footer readable.
6. Settings: change idle timeout value → background app → foreground after timeout → lock screen shown.
7. Idle timeout fires mid-session → lock screen shown → authenticate → return to previous screen.

**Owner:** df-coder  
**Completion Criteria:** All five checks pass. End-to-end smoke completes without crashes or unexpected errors. Emulator/device model identified in return handoff.

---

## Progress Sheet

| Stage | Owner | Status | Completion Criteria | Notes |
| --- | --- | --- | --- | --- |
| 1 — UX Review | df-coder | Not Started | All checklist items reviewed; issues logged with severity | Catalogue only — no fixes in this stage |
| 2 — UX Fixes | df-coder | Not Started | All Stage 1 issues resolved; analyze clean; tests pass | Minimal changes only; no feature additions |
| 3 — Keyboard & Scroll | df-coder | Not Started | All text-input screens scroll correctly with keyboard open; single-line last fields use `TextInputAction.done`; multiline notes field uses `TextInputAction.newline`; analyze/tests pass | LogEntryScreen notes field must NOT use `TextInputAction.done`; keyboard dismiss via tap-outside |
| 4 — Release Build | df-coder | Not Started | `flutter build apk --release` succeeds; ProGuard disabled; `key.properties`+`*.jks` in `android/.gitignore`; no credentials in `build.gradle`; OQ1 signing status noted | `key.properties` credential injection pattern required — see stage |
| 5 — Documentation | df-coder | Not Started | `v1.0.0.md` created and complete; `README.md` updated with build instructions | Create `Documentation/Records/Implementation-Summaries/` if absent |
| 6 — Final Validation | df-coder | Not Started | All five checks pass; end-to-end smoke complete | Emulator/device identified in return handoff |

---

## Acceptance Criteria

1. All Phase 1–6 screens are reachable via normal navigation flows; no dead-end routes.
2. Lock screen `PopScope(canPop: false)` in place; back button does not exit the app from the lock screen.
3. Every list screen has an appropriate empty state widget — no blank screens, no raw `null` renders.
4. Every async screen handles loading, data, and error branches with appropriate UI (indicator, content widget, error message).
5. `LogEntryScreen` notes field and any multi-field form scrolls correctly when the keyboard is open; no field obscured.
6. All text fields have correct `TextInputAction` (`TextInputAction.done` for single-line last fields; `TextInputAction.newline` for multiline notes field); keyboard dismiss via tap-outside works.
7. `flutter analyze` — zero issues.
8. `dart format lib/ test/` — no changes needed.
9. `flutter test` — all tests pass.
10. `flutter build apk --debug` — succeeds.
11. `flutter build apk --release` — succeeds with ProGuard/R8 disabled; signing status noted in return handoff.
12. `Documentation/Records/Implementation-Summaries/v1.0.0.md` exists and is complete.
13. `README.md` updated with prerequisites, build commands, and test commands.
14. End-to-end smoke on emulator completes all seven flows without crashes or unexpected errors.

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| UX review surfaces a medium/high severity bug requiring feature-touching fix | Medium | Medium | Scope-gate fix to minimum change; if new feature work is implied, raise as post-v1 issue |
| Hive adapter classes stripped by R8 in release mode | High | High | Mitigated by `minifyEnabled false` in Stage 4 pre-step |
| `flutter_secure_storage` runtime failure in release mode due to ProGuard | Medium | High | Mitigated by `minifyEnabled false` in Stage 4 pre-step |
| Release signing key not available at Stage 4 | High | Medium | OQ1 open; Stage 4 documents both paths; debug-signed APK acceptable for Phase 7 validation, not for publication |
| `flutter test` regression from Stage 2 UX fixes | Low | Low | Run `flutter test` after every fix batch before proceeding to next issue |

---

## Handoff Target

df-qa — via `Documentation/Handoffs/df-planner_to_df-qa.md`
