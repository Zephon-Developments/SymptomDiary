# Plan: Phase 4 — Symptom Logging

**Created:** 2026-06-02  
**Status:** Approved — Ready for Implementation (pending Phase 1 + 2 + 3)  
**Parent Plan:** `Documentation/Plans/Plan_Initial_App_Delivery.md`  
**Prerequisite:** Phase 1, Phase 2, and Phase 3 complete and approved by df-qa.  
**Active Phase:** Phase 4 — Symptom Logging

---

## Objective

Implement the core daily entry flow: pain level input (0–10), symptom selection from a predefined and custom list, freeform notes, entry save/update, missed entry backdating, quick preset management, and a home screen that surfaces today's entry state and the quick-log action. This phase delivers the primary user-facing feature of the app.

---

## Constraints

- All data written through `DiaryService`, `SymptomService`, and `PresetService` (Phase 2). No direct Hive calls in UI code.
- Providers already stubbed in Phase 2 (`diary_provider.dart`, `symptoms_provider.dart`, `presets_provider.dart`) — Phase 4 replaces stubs with real `AsyncNotifier` implementations.
- No UI hookup was done in Phases 1–3; all screens are currently placeholders.
- Future entries (dates beyond today) are not permitted.
- Rapid delivery and low complexity are explicit product priorities (CODING_STANDARDS).
- Android-first; iOS not in scope for v1.

---

## Assumptions

1. Phase 1 navigation shell, Phase 2 encrypted services, and Phase 3 auth/lock layer are complete before implementation starts.
2. `DiaryServiceHive`, `SymptomServiceHive`, and `PresetServiceHive` are constructor-injected and available via Riverpod providers.
3. The log entry screen (`log_entry_screen.dart`) is the single entry point for creating and editing a `DailyEntry`.
4. Pain level is an integer in range [0, 10] stored on `DailyEntry.painLevel`.
5. Symptom selection is stored as `DailyEntry.symptomIds: List<String>` — a list of `Symptom.id` values (IDs, not names). Phase 2 model field name is `symptomIds`.
6. The predefined symptom list is a hardcoded constant pending OQ1 resolution. A placeholder list is used until OQ1 is confirmed.
7. Custom symptoms (`Symptom(isPredefined: false)`) are stored in the Hive `symptoms` box and appear alongside predefined symptoms in the picker.
8. Presets store a name and a list of **symptom IDs** (`Preset.symptomIds: List<String>`). Applying a preset pre-populates the symptom picker by matching preset symptom IDs against the combined predefined + custom symptom list.
9. Missed entry backdating uses `showDatePicker` restricted to dates ≤ today. The same `LogEntryScreen` is reused for any date.
10. The home screen shows today's entry status (logged/not logged) and provides a quick-log navigation action. Full history display is Phase 5.
11. `AsyncNotifierProvider` is used for all three feature providers (per CODING_STANDARDS — async data from Hive storage).

---

## Open Questions

| ID  | Question | Status |
| --- | -------- | ------ |
| OQ1 | What is the confirmed predefined symptom list? The parent plan defers this to Phase 4 (Assumption 6). A placeholder list is used until confirmed. | Open — coordinator/owner input required before Stage 4 ships |
| OQ2 | Home screen: show a pain level + symptom summary for today's entry, or just a "Logged / Not Logged" indicator with a quick-log CTA? | Open — low risk; either approach is acceptable. Default: summary if logged, CTA if not. |

---

## Package Dependencies

No new packages required. Phase 2 (`hive`, `hive_flutter`, `uuid`) and Phase 3 (`shared_preferences`) dependencies cover all Phase 4 needs.

---

## Directory Structure (additions to Phase 3 scaffold)

```
lib/
  features/
    diary/
      screens/
        home_screen.dart            # Replaces placeholder — today's entry state + quick-log
        log_entry_screen.dart       # Replaces placeholder — full log entry form
      providers/
        diary_provider.dart         # AsyncNotifierProvider.family<DiaryNotifier, DailyEntry?, DateTime> — keyed by date
      widgets/
        pain_level_picker.dart      # 0–10 integer slider with numeric label
        notes_field.dart            # Freeform text input widget
    symptoms/
      screens/
        add_symptom_screen.dart     # Add a new custom symptom
      providers/
        symptoms_provider.dart      # Replace Phase 2 stub — AsyncNotifierProvider<SymptomsNotifier, List<Symptom>>
      widgets/
        symptom_chip_grid.dart      # Wrap layout of selectable symptom chips
    presets/
      screens/
        presets_screen.dart         # List, delete, and apply presets
      providers/
        presets_provider.dart       # Replace Phase 2 stub — AsyncNotifierProvider<PresetsNotifier, List<Preset>>
      widgets/
        preset_list_tile.dart       # Single preset row with apply and delete actions
test/
  features/
    diary/
      diary_provider_test.dart
    symptoms/
      symptoms_provider_test.dart
    presets/
      presets_provider_test.dart
```

---

## Stage Plan

### Stage 1 — Wire Providers

Replace Phase 2 stub providers with real `AsyncNotifier` implementations.

**Provider bootstrapping chain (M1):**

Phase 2's `EncryptedStorage.init()` opens all Hive boxes before `runApp`. Those `Box<T>` references enter the Riverpod graph via `ProviderScope` overrides. Define three box providers:

```dart
final diaryBoxProvider = Provider<Box<DailyEntry>>(
  (ref) => throw UnimplementedError('not initialized'));
final symptomBoxProvider = Provider<Box<Symptom>>(
  (ref) => throw UnimplementedError('not initialized'));
final presetBoxProvider = Provider<Box<Preset>>(
  (ref) => throw UnimplementedError('not initialized'));
```

In `main.dart`, override them at `ProviderScope` startup:

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

Service providers read their box provider via `ref.watch`:

```dart
final diaryServiceProvider = Provider<DiaryServiceHive>(
  (ref) => DiaryServiceHive(ref.watch(diaryBoxProvider)));
```

Apply the same pattern for `symptomServiceProvider` and `presetServiceProvider`.

**DiaryNotifier — family provider (M2):**

`DiaryNotifier` is a **family** provider keyed by `DateTime date`:

- Provider type: `AsyncNotifierProvider.family<DiaryNotifier, DailyEntry?, DateTime>`
- `build(DateTime date)` loads and returns `DailyEntry?` for that date automatically — no `loadForDate` method.
- Action methods on the notifier: `saveEntry(DailyEntry)`, `deleteEntry(DateTime)`.
- Call sites: `ref.watch(diaryProvider(date))` to observe state; `ref.read(diaryProvider(date).notifier)` to call actions.

**SymptomsNotifier and PresetsNotifier:**

- `SymptomsNotifier` (`symptoms_provider.dart`) — `AsyncNotifierProvider<SymptomsNotifier, List<Symptom>>`. Returns **all** symptoms (predefined + custom combined). Methods: `addCustom(String name)`, `deleteCustom(String id)`.

  `build()` merges `kPredefinedSymptoms` with Hive-stored custom symptoms:

  ```dart
  Future<List<Symptom>> build() async {
    final service = ref.watch(symptomServiceProvider);
    final custom = await service.getAllSymptoms(); // Hive-stored custom only
    return [...kPredefinedSymptoms, ...custom];
  }
  ```

  This ensures `symptomsProvider` always returns the full combined list required for ID-to-name resolution in the home screen, `PresetsScreen`, and preset apply.

- `PresetsNotifier` (`presets_provider.dart`) — `AsyncNotifierProvider<PresetsNotifier, List<Preset>>`. Methods: `savePreset(String name, List<String> symptomIds)`, `deletePreset(String id)`.

Both receive their service via `ref.watch(symptomServiceProvider)` / `ref.watch(presetServiceProvider)` in `build()`.

**Owner:** df-coder  
**Completion Criteria:** All three providers load from Hive via their respective services. Box providers wired through `ProviderScope` overrides in `main.dart`. `DiaryNotifier` declared as family provider. No `loadForDate` method. `SymptomsNotifier.build()` merges `kPredefinedSymptoms` + Hive custom symptoms. No direct Hive calls in notifier code.

---

### Stage 2 — Home Screen

Replace the Phase 1 `HomeScreen` placeholder:

- On load: call `ref.watch(diaryProvider(today))` where `today` is `DateTime(now.year, now.month, now.day)`.
- If entry exists: show pain level badge and up to 3 symptom chips as a summary row. Resolve `entry.symptomIds` to display names by also watching `symptomsProvider`: `allSymptoms.where((s) => entry.symptomIds.contains(s.id))`. Tapping opens `LogEntryScreen` for today.
- If no entry: show a "Log Today" prominent CTA button that navigates to `LogEntryScreen` for today.
- "Log Past Entry" text button (secondary) opens a date picker to navigate to `LogEntryScreen` for a past date.

**Owner:** df-coder  
**Completion Criteria:** Home screen reflects today's logged state on load and after returning from `LogEntryScreen`. OQ2 default applied.

---

### Stage 3 — Log Entry Screen — Pain Level and Date

Replace the Phase 1 `LogEntryScreen` placeholder with a form scaffold:

- Accepts a `DateTime date` parameter (defaults to today). Restricted to dates ≤ today.
- On load: pre-populate with existing `DailyEntry` for the date if one exists. The screen watches `ref.watch(diaryProvider(date))` — data loads automatically when the family provider is created for the given date.
- Pain level input: `Slider` (`min: 0`, `max: 10`, `divisions: 10`) with a numeric label showing the current value.
- Header shows the date with a change-date icon (only enabled if the screen was launched from the "Log Past Entry" flow).

**Owner:** df-coder  
**Completion Criteria:** Log entry screen opens for today and for past dates. Pain level slider is interactive and shows current value. Existing entry pre-populates on open.

---

### Stage 4 — Symptom Picker

Add symptom selection to `LogEntryScreen`:

- `SymptomChipGrid` widget: displays all symptoms (predefined + custom) as selectable `FilterChip` widgets in a `Wrap` layout.
- Predefined list: hardcoded constant in `lib/features/symptoms/symptom_data.dart`. Use a `const List<Symptom>` with short, stable string IDs and `isPredefined: true`:

```dart
const List<Symptom> kPredefinedSymptoms = [
  Symptom(id: 'fatigue',    name: 'Fatigue',             isPredefined: true),
  Symptom(id: 'headache',   name: 'Headache',            isPredefined: true),
  Symptom(id: 'nausea',     name: 'Nausea',              isPredefined: true),
  Symptom(id: 'dizziness',  name: 'Dizziness',           isPredefined: true),
  Symptom(id: 'joint_pain', name: 'Joint pain',          isPredefined: true),
  Symptom(id: 'breath',     name: 'Shortness of breath', isPredefined: true),
  Symptom(id: 'swelling',   name: 'Swelling',            isPredefined: true),
  Symptom(id: 'numbness',   name: 'Numbness',            isPredefined: true),
];
```

These are not stored in Hive. Chip selection stores `Symptom.id` values into `DailyEntry.symptomIds`. Pending OQ1: if the confirmed list differs, only this constant changes.
- All symptoms from `symptomsProvider` (predefined + custom, merged in `SymptomsNotifier.build()`) render as `FilterChip` widgets. The chip grid simply renders the full list from `ref.watch(symptomsProvider)`.
- "Add symptom" chip (with `+` icon) at the end of the list navigates to `AddSymptomScreen`.
- `AddSymptomScreen`: single text field + Save button. Calls `symptomsNotifier.addCustom(name)` on save. Navigates back.

**Owner:** df-coder  
**Completion Criteria:** Symptom chips render; selections update entry state. Custom symptom added via `AddSymptomScreen` appears in the picker immediately. Predefined list is the placeholder constant.

---

### Stage 5 — Notes Field and Save Flow

Add the notes input and wire the save action:

- `NotesField`: multiline `TextField` with a character hint. Binds to `DailyEntry.notes`.
- "Save" button (or FAB): calls `diaryNotifier.saveEntry(entry)`. On success: pops back to the home screen (or previous screen).
- After save: the notifier calls `state = AsyncData(entry)` to update its own state immediately. The home screen's `ref.watch(diaryProvider(today))` receives the updated state automatically — no `ref.invalidate` call needed.
- Validation: pain level is always set (default 0 if untouched). No other mandatory fields.

**Owner:** df-coder  
**Completion Criteria:** Save button writes entry to Hive via `DiaryNotifier`. Navigating back to home screen shows updated entry state. Edit flow (opening existing entry) updates the record rather than creating a duplicate.

---

### Stage 6 — Preset Management

Implement preset save and management:

- On `LogEntryScreen`: "Save as Preset" action (icon button in app bar or context menu). Shows a dialog with a name field. Calls `presetsNotifier.savePreset(name, selectedSymptomIds)` where `selectedSymptomIds` is the current list of selected `Symptom.id` values.
- `PresetsScreen` (accessible from the log entry screen app bar and/or from Settings): lists saved presets. Each row shows name + symptom summary. To display symptom names, `PresetsScreen` must watch `symptomsProvider` and resolve `preset.symptomIds` to names: `allSymptoms.where((s) => preset.symptomIds.contains(s.id)).map((s) => s.name)`. Delete icon calls `presetsNotifier.deletePreset(id)`.
- Empty state for `PresetsScreen` when no presets exist.
- Register `/presets` as a named route in `app.dart`'s route map during this stage. Navigation from `LogEntryScreen` and Settings uses `Navigator.pushNamed(context, '/presets')`.

**Owner:** df-coder  
**Completion Criteria:** Preset save dialog works. `PresetsScreen` lists saved presets. Delete removes from list and Hive.

---

### Stage 7 — Apply Preset Flow

Add one-tap preset application on `LogEntryScreen`:

- "Apply Preset" button (text button or icon in the symptom section). Opens a bottom sheet or dialog listing saved presets from `PresetsNotifier`.
- Tapping a preset replaces the current symptom selection with the preset's symptom list.
- Empty state shown if no presets exist (with a prompt to save one first).

**Owner:** df-coder  
**Completion Criteria:** Applying a preset updates the symptom chip selection immediately. If a preset contains a custom symptom that was deleted, skip the missing symptom silently.

---

### Stage 8 — Provider and Widget Tests

Tests covering:

- `DiaryNotifier`: load returns null for no entry; save creates entry; save for same date updates entry (not duplicate); delete removes entry.
- `SymptomsNotifier`: `build()` returns predefined list prepended to Hive custom symptoms (`[...kPredefinedSymptoms, ...custom]`); `addCustom` persists; `deleteCustom` removes; predefined list not stored in Hive (it is a constant).
- `PresetsNotifier`: save persists; delete removes; apply returns correct symptom list.
- Widget test for `LogEntryScreen`: loads existing entry; save calls notifier; back navigation from save.
- Widget test for `HomeScreen`: shows CTA when no entry; shows summary when entry exists.

All provider tests use `ProviderScope(overrides: [...])` with mock services (per Phase 2 constructor-injection pattern).

Note: `DiaryNotifier` is a family provider. Tests use family call syntax: `container.read(diaryProvider(date).notifier).saveEntry(...)` where `container` is a `ProviderContainer` with the appropriate box provider overrides.

Note: `HomeScreen` widget test for the summary state (AC9) must override both `diaryProvider(today)` with a `DailyEntry` containing `symptomIds` **and** `symptomsProvider` with a matching `List<Symptom>`, so that ID-to-name resolution works correctly in the test.

**Owner:** df-coder  
**Completion Criteria:** `flutter test` passes; all critical save/load paths covered.

---

### Stage 9 — Validation

1. `dart format lib/ test/`
2. `flutter analyze` — zero issues
3. `flutter test` — all tests pass
4. `flutter build apk --debug` — succeeds
5. Manual smoke on emulator: open app → authenticate → home (no entry) → log today → pain level + symptoms + notes → save → home reflects entry → tap entry → edit → save → log past entry → preset save → preset apply → presets screen delete.

**Owner:** df-coder  
**Completion Criteria:** All five checks pass. df-coder handoff includes OQ1 status (placeholder list used if unresolved).

---

## Progress Sheet

| Stage | Owner | Status | Completion Criteria | Notes |
| --- | --- | --- | --- | --- |
| 1 — Wire Providers | df-coder | Not Started | All three notifiers load from Hive and expose methods | SymptomsNotifier.build() merges kPredefinedSymptoms + Hive custom symptoms (cycle 2 re-review M1). Box providers + ProviderScope overrides (M1). DiaryNotifier family provider (M2). Phase 2 field names aligned (cycle 1 re-review M1). |
| 2 — Home Screen | df-coder | Not Started | Today's entry state shown; quick-log CTA works | OQ2 default applied. ID-to-name resolution via symptomsProvider added (cycle 1 re-review L1). |
| 3 — Log Entry Screen (pain + date) | df-coder | Not Started | Slider interactive; existing entry pre-populates | Date restricted to ≤ today |
| 4 — Symptom Picker | df-coder | Not Started | Chips render; custom symptom add works | Chip grid renders full list from symptomsProvider (predefined + custom); kPredefinedSymptoms constant is List<Symptom> with IDs. Pending OQ1 for final list. |
| 5 — Notes + Save Flow | df-coder | Not Started | Save writes to Hive; home updates on return | state = AsyncData(entry) after save; no ref.invalidate (L2) |
| 6 — Preset Management | df-coder | Not Started | Save preset works; list and delete work; /presets route registered | PresetsScreen watches symptomsProvider for name display (cycle 2 re-review L1). /presets route in app.dart. |
| 7 — Apply Preset | df-coder | Not Started | One-tap applies symptom selection; missing symptoms skipped | Bottom sheet or dialog |
| 8 — Provider and Widget Tests | df-coder | Not Started | flutter test passes; save/load/delete paths covered | SymptomsNotifier build() merge test added. HomeScreen AC9 widget test overrides both diaryProvider(today) and symptomsProvider (cycle 2 re-review L2). |
| 9 — Validation | df-coder | Not Started | format/analyze/test/build/smoke all pass | Note OQ1 status in handoff |

---

## Acceptance Criteria

1. Pain level (0–10) slider is present and interactive on the log entry screen.
2. Pain level defaults to 0 if the user does not interact with the slider.
3. Today's entry auto-populates with existing data when the log entry screen is opened for today.
4. Symptom chips render the predefined list (placeholder constant pending OQ1) plus any user-defined custom symptoms.
5. Custom symptom can be added via `AddSymptomScreen` and immediately appears in the chip list.
6. Freeform notes field accepts and saves multiline text.
7. Saving an entry creates a new `DailyEntry` or updates the existing one for that date (no duplicates).
8. Home screen shows a "Log Today" CTA when no entry exists for today.
9. Home screen shows today's pain level and symptom summary when an entry exists.
10. Missed entry backdating: user can open a date picker from the home screen and create/edit an entry for any past date.
11. Future dates are not selectable in the date picker.
12. Preset can be saved with a name from the current symptom selection.
13. Preset can be applied to pre-populate the symptom picker on the log entry screen.
14. Preset can be deleted from `PresetsScreen`.
15. Applying a preset with a deleted custom symptom does not crash (missing symptom silently skipped).
16. All three feature providers (`DiaryNotifier`, `SymptomsNotifier`, `PresetsNotifier`) have unit tests covering create, read, update, delete paths.
17. Widget tests cover the log entry save flow and home screen state transitions.
18. `flutter analyze` passes with zero issues.
19. `flutter test` passes with all tests green.
20. `flutter build apk --debug` succeeds.
21. Manual smoke test on emulator confirms full log, edit, preset, and backdating flows.

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| OQ1 (predefined symptom list) unresolved before Stage 4 | High | Low | Placeholder `kPredefinedSymptoms` constant used; only the constant changes when OQ1 resolves |
| Home screen not refreshing after save | Medium | Low | After `saveEntry` succeeds, call `state = AsyncData(entry)` directly on the notifier. `ref.watch(diaryProvider(today))` on the home screen receives the new state automatically. Verify in widget test. |
| Preset apply with deleted custom symptoms causes exception | Low | Medium | Stage 7 — filter missing symptoms before applying; covered by AC15 |
| Widget test setup for screens requiring auth state (Phase 3 lock) | Medium | Low | Override `authProvider` with `authenticated` state in `ProviderScope` for widget tests |
| `EncryptedStorage.init()` return type unspecified in Phase 2 | Medium | Medium | Phase 2 must implement `EncryptedStorage.init()` to return a named-property record (e.g., `EncryptedBoxes` data class) with `diaryBox`, `symptomBox`, `presetBox` fields. Phase 4 bootstrap code depends on this contract. df-coder must establish this return type when implementing Phase 2. |

---

## Handoff Target

**df-coder** — implement Phase 4 following `Documentation/Handoffs/df-qa_to_df-coder.md`.

All cycle findings resolved. Implementation notes N1 and N2 are in the df-coder handoff:
- N1: `SymptomsNotifier.addCustom`/`deleteCustom` must update `state = AsyncData(newList)` after persisting (same pattern as DiaryNotifier Stage 5). Required for AC5 (symptom appears immediately).
- N2: Stage 8 “PresetsNotifier: apply returns correct symptom list” — implement as a widget test (apply is a UI-level action in Stage 7; no `apply` method exists on `PresetsNotifier`). See Stage 7 for filter logic.
