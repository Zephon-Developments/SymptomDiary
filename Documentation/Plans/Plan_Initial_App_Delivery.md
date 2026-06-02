# Plan: Initial App Delivery

**Created:** 2026-06-02  
**Status:** Approved — Top-Level Phase Roadmap  
**Active Phase:** Phase 1 — Project Scaffold

---

## Objective

Deliver a working Android-first Flutter application that allows a single user to record daily pain levels, symptoms, and freeform notes; review their symptom history; generate printable reports for healthcare providers; and do all of this with strong local encryption and phone reauthentication on launch.

---

## Constraints

- Android-first Flutter application.
- Local-first, offline-first. No backend or sync in v1.
- Single-user. No multi-profile support.
- All stored health data must be encrypted at rest. No non-settings data in plaintext.
- Phone reauthentication (biometric or PIN via device credentials) required on launch.
- Idle timeout required (configurable, default 5 minutes).
- Rapid delivery and low complexity are explicit product priorities.
- All workflow artifacts must remain inside this repository under `Documentation/`.

---

## Assumptions

1. Target platform is Android. iOS is not in scope for v1.
2. Device credential authentication (biometric/PIN) will use the `local_auth` package. Custom PIN management is out of scope.
3. Encrypted local storage will use Hive with AES encryption. If Hive proves unsuitable, sqflite with SQLCipher is the fallback.
4. "Printable report" means a PDF file the user can export or share via the Android share sheet. Native print support is a stretch goal.
5. "Quick presets" means saved symptom combinations the user can apply in one tap during logging.
6. The predefined symptom list will be defined during Phase 4. The exact list is deferred.
7. "Missed entries" means the user can create or edit entries for any past date, but cannot create future-dated entries.
8. Settings (e.g., idle timeout duration) are stored unencrypted as they contain no sensitive health data.
9. Minimum Android SDK is API 23 (Android 6.0). This is required by the `local_auth` package for biometric support and is the practical floor for all v1 package dependencies. Target SDK and maximum supported Android version is Android 12 (API 31). compileSdkVersion is 31. Android 13 and above is not in scope for v1.
10. No localization or accessibility targets are defined for v1 beyond standard Flutter defaults.
11. The idle timeout behavior implements a lock screen (return to the authentication screen) rather than app termination. This is an intentional and confirmed deviation from the literal "exit" wording in user requirements. The idle timeout clock pauses while the lock screen is displayed and resets on successful authentication.

---

## Phased Plan

### Phase 1 — Project Scaffold

Establish the Flutter project, directory structure, navigation shell, and build pipeline. No feature logic. Validates that the project builds and installs on Android.

**Deliverables:**

- Flutter project initialised with the folder structure from CODING_STANDARDS.
- State management: Riverpod (`flutter_riverpod`) selected, documented in `Documentation/Reference/CODING_STANDARDS.md`, added to `pubspec.yaml`, and provider structure scaffolded.
- Bottom navigation or drawer-based shell with placeholder screens.
- App theming applied (colours, typography).
- `flutter analyze` passing clean.
- Successful Android debug build.

---

### Phase 2 — Data Model and Encrypted Persistence

Define the core typed models and implement an encrypted local storage layer with a CRUD service interface. No UI hookup; logic is independently testable.

**Deliverables:**

- Typed models: `DailyEntry`, `Symptom`, `PainLevel`.
- Support for both predefined and user-defined custom symptoms.
- Encrypted Hive database (or approved fallback).
- `DiaryService` with create, read, update, delete for entries.
- Unit tests for models and service layer.
- No stored data exists in plaintext (verified by test).

---

### Phase 3 — Security Layer

Implement phone reauthentication on launch and idle timeout. The app must lock itself and require reauthentication before granting access to data.

**Deliverables:**

- `local_auth` integration: biometric or device PIN on cold launch.
- Lock screen UI shown on launch and after idle timeout.
- `IdleTimeoutService` with configurable duration (default 5 min). Idle clock pauses while the lock screen is visible and resets on successful authentication.
- App returns to lock screen on timeout regardless of current screen.
- Settings screen with idle timeout duration option (unencrypted).
- Graceful handling when device has no enrolled credentials (show clear user message).

---

### Phase 4 — Symptom Logging

Implement the core daily entry flow: pain level, symptom selection, freeform notes, backdating for missed entries, and quick presets.

**Deliverables:**

- Today's entry screen with 0–10 pain level input.
- Symptom picker: predefined list selectable via chips/toggles.
- Add custom symptom flow.
- Freeform notes text field.
- Save and update entry.
- Missed entry flow: create or edit an entry for any past date.
- Quick preset management: save a named symptom combination and apply it in one tap.
- Home screen surfaces today's entry state and quick-log action.

---

### Phase 5 — History Review

Provide a clear, navigable view of past entries so the user can identify patterns and trends.

**Deliverables:**

- Calendar or timeline view of entries with visual pain level indicators.
- Symptom indicator icons per day.
- Entry detail view (pain level, symptoms, notes, date).
- Filter and sort: by date range, by symptom, by pain level threshold.
- Empty state messaging for days with no entry.

---

### Phase 6 — Printable Reports

Allow the user to generate and export a report of their symptom history for sharing with healthcare providers.

**Deliverables:**

- Report configuration screen: date range selection.
- PDF report generation using `pdf` or `printing` package.
- Report includes pain levels, symptoms, and notes per day in a readable format.
- Android share sheet integration for export.
- Report generation does not store any unencrypted intermediate file.

---

### Phase 7 — Polish and Release Preparation

UX review, edge case hardening, and release build preparation. No new features.

**Deliverables:**

- UX review pass: navigation flows, empty states, error states.
- Keyboard handling and scroll behaviour confirmed on target devices.
- Release-mode Android build passes.
- `flutter analyze` clean, no unaddressed warnings.
- `Documentation/Records/Implementation-Summaries/` updated with a completion summary.
- README updated with setup and build instructions.

---

## Progress Sheet

| Stage | Owner | Status | Completion Criteria | Notes |
| --- | --- | --- | --- | --- |
| Phase 1 — Project Scaffold | df-coder | Not Started | Clean analyze, successful Android debug build, nav shell present, state management documented | Riverpod selected; min SDK API 23 / target SDK API 31 (Android 12); lock behavior confirmed. Detailed Phase 1 plan required before coding begins. |
| Phase 2 — Data Model & Encrypted Persistence | df-coder | Not Started | Models defined, encrypted storage working, unit tests passing, no plaintext data | Hive or sqflite+SQLCipher TBD |
| Phase 3 — Security Layer | df-coder | Not Started | Lock screen on launch and idle, configurable timeout, graceful no-credentials handling | Depends on Phase 1 + Phase 2 |
| Phase 4 — Symptom Logging | df-coder | Not Started | Full entry flow working end-to-end including backdating and presets | Predefined symptom list to be confirmed |
| Phase 5 — History Review | df-coder | Not Started | Calendar/timeline view, detail view, filter/sort working | Depends on Phase 4 |
| Phase 6 — Printable Reports | df-coder | Not Started | PDF generated and shareable, no unencrypted intermediate files | Approved — ready for implementation after Phase 5 |
| Phase 7 — Polish & Release Prep | df-coder | Not Started | Release build passes, analyze clean, docs updated | Approved — ready for implementation after Phase 6 |

---

## Acceptance Criteria

1. A user can log a pain level (0–10), one or more symptoms (predefined or custom), and freeform notes for any day.
2. A user can create an entry for a missed past day.
3. A user can apply a saved quick preset to pre-populate symptoms during logging.
4. The app requires phone credential authentication on every cold launch.
5. The app locks and returns to the authentication screen after the configured idle period (default 5 min).
6. All health data is encrypted at rest; no entry data exists as plaintext on the device.
7. A user can review a calendar or timeline of past entries showing pain levels and symptom indicators.
8. A user can filter or sort history by date, symptom, or pain level.
9. A user can generate a PDF report for a chosen date range and share it via the Android share sheet.
10. The release-mode Android APK builds without errors.

---

## Validation Approach

- **Per phase**: `dart format`, `flutter analyze`, narrowest relevant `flutter test` scope. Any unrun checks noted in handoff.
- **Phase 2**: Unit tests must demonstrate encrypted write and read round-trip; assert no plaintext file exists on disk.
- **Phase 3**: Manual verification of lock-on-launch and idle timeout on a device or emulator.
- **Phase 4–6**: Widget and integration tests for primary flows. Manual verification on Android emulator.
- **Phase 7**: Release-mode build (`--release`) must complete; `flutter analyze` must be clean.

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| `local_auth` behaves differently on emulator vs real device for biometrics | Medium | Medium | Test on real device early in Phase 3; document emulator workaround |
| Hive encrypted box has performance issues with large history | Low | Low | Phase 2 validates with representative data; fallback to sqflite+SQLCipher |
| PDF generation library incompatible with target Android SDK | Low | Medium | Spike library choice in Phase 6 before full implementation |
| Idle timeout service interrupted by Android OS background restrictions | Medium | High | Investigate foreground vs background behaviour in Phase 3; use `AppLifecycleState` listener |
| Predefined symptom list undefined | High | Low | Blocked only on Phase 4 start; coordinator to confirm list before Phase 4 begins |

---

## Deferred Items

The following are explicitly out of scope for v1 and should not be designed or implemented unless scope changes:

- iOS support
- Backend sync or cloud backup
- Multi-user or multi-profile support
- Push notifications or reminders
- Symptom analytics or trend graphs beyond basic history display
- Custom print layout or native print dialog
- Accessibility audit beyond Flutter defaults
- Localization / internationalisation

---

## Open Questions — Status (updated 2026-06-02)

1. **Minimum Android SDK version** — **Closed.** Minimum SDK: API 23 (Android 6.0). Target/maximum SDK: API 31 (Android 12). compileSdkVersion: 31. See Assumption 9.
2. **Predefined symptom list content** — Open. Coordinator to confirm before Phase 4 begins. Not a Phase 1–3 blocker.
3. **Idle timeout clock behaviour** — **Closed.** QA recommendation adopted: idle clock runs only while the app is in the active (unlocked) state; it pauses while the lock screen is visible and resets on successful authentication. See Assumption 11 and Phase 3 deliverables.
4. **Target device or emulator API level for test validation** — **Closed.** Target API 31 (Android 12). See Assumption 9.
5. **Quick presets v1 vs v1.1** — **Closed by QA review.** User requirements and copilot-instructions.md both explicitly reference quick presets as a v1 feature. Phase 4 scope is confirmed.

**Additional QA finding — idle timeout interpretation:** User requirements state the app should "exit" if left unattended. The plan implements a lock-screen approach (return to authentication) rather than app termination. This is better security and UX practice. **Confirmed acceptable by product owner.**
