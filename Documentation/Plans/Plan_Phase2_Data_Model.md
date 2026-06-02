# Plan: Phase 2 — Data Model and Encrypted Persistence

**Created:** 2026-06-02  
**Status:** Approved — Phase 2 Ready for Implementation (pending Phase 1 completion)  
**Parent Plan:** `Documentation/Plans/Plan_Initial_App_Delivery.md`  
**Prerequisite:** Phase 1 scaffold complete and passed df-qa review.  
**Active Phase:** Phase 2 — Data Model and Encrypted Persistence

---

## Objective

Define the core typed models and implement an encrypted local storage layer with a CRUD service interface. No UI hookup; all logic is independently testable. Deliverable is a tested persistence foundation that Phases 3–6 build on.

---

## Constraints

- All health data (diary entries, symptoms, presets) stored in encrypted Hive boxes. No plaintext.
- Encryption key stored in `flutter_secure_storage`; never logged or exposed.
- `DiaryService` must be an abstract interface; concrete implementation is Hive-backed.
- Providers defined here are stubs; full UI wiring is Phase 4.
- The predefined symptom list is **not** implemented here (deferred to Phase 4, per parent plan Assumption 6). A stub returning `[]` is acceptable.
- Rapid delivery and low complexity are explicit priorities (CODING_STANDARDS).

---

## Assumptions

1. Phase 1 directory structure is in place before implementation starts.
2. `flutter_secure_storage` key generation and retrieval is sufficient for AES key management. No custom key derivation.
3. `uuid` package (`^4.5.1`) is used for ID generation.
4. Hive box names: `diary_entries`, `symptoms`, `presets`.
5. `DailyEntry.painLevel` is stored as `int` (0–10). `PainLevel` is a lightweight utility class, not a separate Hive type.
6. Entries are one-per-day. `getEntryForDate` enforces this; `saveEntry` upserts by date.
7. Settings data (e.g., idle timeout) is **not** encrypted; out of scope for this phase.
8. Unit tests use `Hive.init(tempDir)` with a real temporary directory rather than a mock, to test actual adapter round-trips.
9. `DailyEntry.date` is normalized to midnight local time before storage using `DateTime(d.year, d.month, d.day)`. All date comparisons use this normalized form. This prevents off-by-one errors across midnight or timezone changes.
10. The Hive box key for `diary_entries` is the normalized date string in `YYYY-MM-DD` format (e.g. `'2026-06-02'`). This makes `saveEntry` upsert and `getEntryForDate` O(1). `DailyEntry.id` remains a UUID for use as a domain identifier but is not the box key.
11. `DiaryServiceHive` and `SymptomServiceHive` (and `PresetServiceHive`) accept their Hive `Box<T>` via constructor injection. This keeps services testable without a running Flutter app.
12. If `EncryptedStorage.init()` cannot decrypt the existing box (key mismatch — e.g., after uninstall/reinstall), it deletes the corrupted boxes and reinitialises fresh. A user-visible warning is deferred to Phase 3.

---

## Open Questions

| ID  | Question                                                                                    | Status                                        |
| --- | ------------------------------------------------------------------------------------------- | --------------------------------------------- |
| OQ1 | Include `Preset` model in Phase 2 (QA LOW from prior cycle) or keep deferred to Phase 4?  | **Resolved YES** — Preset included in Phase 2. Preset *UI* and "apply preset" flow remain Phase 4. |

---

## Package Dependencies

Additions to `pubspec.yaml`:

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

---

## Data Model Design

### `DailyEntry` (typeId: 0)

**Box key:** normalized date string `'YYYY-MM-DD'` (e.g. `entry.date.toIso8601String().substring(0, 10)`). Enables O(1) upsert and date lookup. `id` is the domain UUID, not the box key.

| Field         | Type            | HiveField | Notes                                                          |
| ------------- | --------------- | --------- | -------------------------------------------------------------- |
| `id`          | `String`        | 0         | UUID v4 — domain identifier, not the Hive box key             |
| `date`        | `DateTime`      | 1         | Normalized to midnight local: `DateTime(y, m, d)`. One/day.   |
| `painLevel`   | `int`           | 2         | 0–10 inclusive                                                 |
| `symptomIds`  | `List<String>`  | 3         | IDs referencing `Symptom.id`                                   |
| `notes`       | `String?`       | 4         | Nullable freeform text                                         |
| `createdAt`   | `DateTime`      | 5         |                                                                |
| `updatedAt`   | `DateTime`      | 6         |                                                                |

### `Symptom` (typeId: 1)
| Field          | Type      | HiveField | Notes                                      |
| -------------- | --------- | --------- | ------------------------------------------ |
| `id`           | `String`  | 0         | Short string for predefined; UUID for custom |
| `name`         | `String`  | 1         |                                            |
| `isPredefined` | `bool`    | 2         | Distinguishes built-in from user-created   |

### `Preset` (typeId: 2 — included; OQ1 resolved Yes)

**Box key:** `id` (UUID v4).

| Field               | Type            | HiveField | Notes              |
| ------------------- | --------------- | --------- | ------------------ |
| `id`                | `String`        | 0         | UUID v4; box key   |
| `name`              | `String`        | 1         |                    |
| `symptomIds`        | `List<String>`  | 2         |                    |
| `defaultPainLevel`  | `int?`          | 3         | Optional default   |

### `PainLevel` (utility class — no Hive type)
Simple value class with a single `int value` (0–10) and a static `isValid(int v)` guard. Used at the domain layer; `DailyEntry` stores the raw `int`.

---

## Directory Structure (additions to Phase 1 scaffold)

```
lib/
  core/
    storage/
      encrypted_storage.dart      # Key management + Hive init
  features/
    diary/
      models/
        daily_entry.dart
        daily_entry.g.dart        # generated
        pain_level.dart           # utility class only
      providers/
        diary_provider.dart       # stub AsyncNotifierProvider
      services/
        diary_service.dart        # abstract interface
        diary_service_hive.dart   # Hive implementation (constructor-injected Box)
    symptoms/
      models/
        symptom.dart
        symptom.g.dart            # generated
        preset.dart
        preset.g.dart             # generated
      providers/
        symptoms_provider.dart    # stub AsyncNotifierProvider
      services/
        symptom_service.dart
        symptom_service_hive.dart # constructor-injected Box
        preset_service.dart       # abstract interface
        preset_service_hive.dart  # constructor-injected Box
test/
  features/
    diary/
      diary_service_test.dart
      daily_entry_adapter_test.dart
    symptoms/
      symptom_service_test.dart
      symptom_adapter_test.dart
      preset_service_test.dart
      preset_adapter_test.dart
```

---

## Stage Plan

### Stage 1 — Add Dependencies

Add `hive`, `hive_flutter`, `flutter_secure_storage`, `uuid` to `pubspec.yaml` dependencies.  
Add `hive_generator`, `build_runner` to `dev_dependencies`.  
Run `flutter pub get` and resolve any version conflicts.

**Owner:** df-coder  
**Completion Criteria:** `pubspec.yaml` updated; `flutter pub get` succeeds with no version conflicts.

---

### Stage 2 — Define Typed Models

Create `DailyEntry`, `Symptom`, and `Preset` in their respective feature directories.  
Apply `@HiveType(typeId: n)` and `@HiveField(n)` annotations.  
Create `PainLevel` utility class (no Hive annotation).

**Owner:** df-coder  
**Completion Criteria:** Model files created; `flutter analyze` passes; no generated files yet.

---

### Stage 3 — Code Generation

Run:
```
dart run build_runner build --delete-conflicting-outputs
```
Commit all generated `.g.dart` adapter files.

**Owner:** df-coder  
**Completion Criteria:** All `.g.dart` files generated, compile cleanly, and are committed.

---

### Stage 4 — Encrypted Storage Initialisation

Before implementation: follow the `flutter_secure_storage` Android setup instructions for the pinned version. Verify no additional `AndroidManifest.xml` entries are required.

Implement `EncryptedStorage` in `lib/core/storage/encrypted_storage.dart`:

- On first run: generate a 32-byte AES key and persist via `flutter_secure_storage`.
- On subsequent runs: retrieve the key from `flutter_secure_storage`.
- **Key mismatch handling:** if decryption fails (e.g. post-uninstall reinstall), catch the exception, delete the corrupted boxes, and reinitialise fresh. Log the event. Full recovery UX is deferred to Phase 3.
- Register all generated Hive adapters.
- Open encrypted Hive boxes: `diary_entries`, `symptoms`, `presets`.
- Return initialised `Box<T>` references for injection into service constructors.

Update `main.dart`:
```dart
await WidgetsFlutterBinding.ensureInitialized();
final boxes = await EncryptedStorage.init();
runApp(ProviderScope(
  overrides: [
    // box overrides for service providers defined in Stages 5 & 6
  ],
  child: const MyApp(),
));
```

**Owner:** df-coder  
**Completion Criteria:** App boots with Hive initialised. Key in `flutter_secure_storage`. Key mismatch is caught and handled. Boxes returned for constructor injection.

---

### Stage 5 — DiaryService

Define abstract interface in `lib/features/diary/services/diary_service.dart`:

```dart
abstract class DiaryService {
  Future<void> saveEntry(DailyEntry entry);
  Future<DailyEntry?> getEntry(String id);
  Future<DailyEntry?> getEntryForDate(DateTime date);
  Future<List<DailyEntry>> getAllEntries(); // date descending
  Future<void> deleteEntry(String id);
}
```

Implement `DiaryServiceHive` in `diary_service_hive.dart`.

- Constructor: `DiaryServiceHive(Box<DailyEntry> box)` — box is injected, not fetched internally.
- Box key is the normalized date string: `entry.date.toIso8601String().substring(0, 10)` (e.g. `'2026-06-02'`).
- `saveEntry` upserts using `box.put(dateKey, entry)` — O(1).
- `getEntryForDate` uses `box.get(dateKey)` — O(1).
- `getAllEntries` returns `box.values.toList()` sorted by `date` descending.
- `deleteEntry` removes by id: scan once with `box.values.firstWhere(...)`, then `box.delete(dateKey)`.

Add stub `AsyncNotifierProvider<DiaryNotifier, List<DailyEntry>>` in `diary_provider.dart` (per CODING_STANDARDS preference for `AsyncNotifierProvider`).

**Owner:** df-coder  
**Completion Criteria:** Interface and Hive implementation compile cleanly. Box injected via constructor. Stub provider registered.

---

### Stage 6 — SymptomService

Define and implement `SymptomService`:

```dart
abstract class SymptomService {
  Future<List<Symptom>> getPredefinedSymptoms(); // stub returns []
  Future<List<Symptom>> getAllSymptoms();         // predefined + custom combined
  Future<void> saveCustomSymptom(Symptom symptom);
  Future<void> deleteCustomSymptom(String id);   // no-op (not throw) if id not found
}
```

`SymptomServiceHive` constructor: `SymptomServiceHive(Box<Symptom> box)` — box injected.

Add `PresetService` abstract interface and `PresetServiceHive(Box<Preset> box)` implementation:

```dart
abstract class PresetService {
  Future<List<Preset>> getAllPresets();
  Future<void> savePreset(Preset preset);
  Future<void> deletePreset(String id);
}
```

Add stub `AsyncNotifierProvider<SymptomsNotifier, List<Symptom>>` in `symptoms_provider.dart`.

**Owner:** df-coder  
**Completion Criteria:** Both interfaces and Hive implementations compile cleanly. Boxes injected via constructors. `getPredefinedSymptoms` returns `[]`. Symptoms provider stub registered.

---

### Stage 7 — Unit Tests

Create tests covering:

- `DailyEntry` Hive adapter round-trip (write to temp box, read back, assert equality).
- `Symptom` Hive adapter round-trip.
- `DiaryServiceHive`: `saveEntry`, `getEntry`, `getEntryForDate`, `getAllEntries` (order), `deleteEntry`.
- `SymptomServiceHive`: `saveCustomSymptom`, `getAllSymptoms`, `deleteCustomSymptom`.
- `PainLevel.isValid`: edge cases 0, 10, -1, 11.
- If OQ1 Yes: `Preset` adapter round-trip and `PresetServiceHive` CRUD.

**Constructor injection pattern:** all service tests construct the service with an injected, opened box.

Test setup pattern:
```dart
late Directory tempDir;
late Box<DailyEntry> entryBox; // (and equivalent for other services)

setUp(() async {
  tempDir = await Directory.systemTemp.createTemp();
  Hive.init(tempDir.path);
  // Register adapters before opening boxes
  if (!Hive.isAdapterRegistered(0)) Hive.registerAdapter(DailyEntryAdapter());
  if (!Hive.isAdapterRegistered(1)) Hive.registerAdapter(SymptomAdapter());
  if (!Hive.isAdapterRegistered(2)) Hive.registerAdapter(PresetAdapter());
  entryBox = await Hive.openBox<DailyEntry>('diary_entries');
});
tearDown(() async {
  await Hive.deleteFromDisk();
  tempDir.deleteSync(recursive: true);
});
```

Service under test constructed as: `final service = DiaryServiceHive(entryBox);`

Additional test cases required:
- `saveEntry` called twice for the same date → only one entry in box (upsert-by-date).
- `deleteCustomSymptom` called with a non-existent ID → completes without throwing.
- `getEntryForDate` on a date with no entry → returns `null`.

**Owner:** df-coder  
**Completion Criteria:** `flutter test` passes with all test cases green.

---

### Stage 8 — Validation

1. `dart format lib/ test/`
2. `flutter analyze` — zero issues
3. `flutter test` — all tests pass
4. `flutter build apk --debug` — succeeds

**Owner:** df-coder  
**Completion Criteria:** All four checks pass. df-coder handoff includes a brief output summary for each check.

---

## Progress Sheet

| Stage | Owner    | Status      | Completion Criteria                                          | Notes                                  |
| ----- | -------- | ----------- | ------------------------------------------------------------ | -------------------------------------- |
| 1 — Add Dependencies          | df-coder | Not Started | pubspec.yaml updated; pub get succeeds                      |                                        |
| 2 — Typed Models              | df-coder | Not Started | Model files created; analyze passes; no generated files yet | Preset included (OQ1 resolved Yes)     |
| 3 — Code Generation           | df-coder | Not Started | .g.dart files generated and committed                       |                                        |
| 4 — Encrypted Storage Init    | df-coder | Not Started | App boots; Hive boxes open; key in secure storage           |                                        |
| 5 — DiaryService              | df-coder | Not Started | Interface + impl + stub provider; box injected via constructor |                                     |
| 6 — SymptomService            | df-coder | Not Started | Interface + impl + PresetService + stub provider; boxes injected |                               |
| 7 — Unit Tests                | df-coder | Not Started | flutter test passes; all cases green                        |                                        |
| 8 — Validation                | df-coder | Not Started | format/analyze/test/build all clean                         |                                        |

---

## Acceptance Criteria

1. `pubspec.yaml` includes `hive`, `hive_flutter`, `flutter_secure_storage`, `uuid`, `hive_generator`, `build_runner` with no version conflicts.
2. `DailyEntry`, `Symptom`, and `Preset` have valid, non-colliding Hive type IDs (0, 1, 2 respectively).
3. All `.g.dart` adapter files are committed to the repository.
4. The AES encryption key is stored exclusively in `flutter_secure_storage`. It does not appear in any Hive box file, log output, or source code.
5. `EncryptedStorage.init()` is called before `runApp` in `main.dart`. Key mismatch (decrypt failure) is caught and handled by deleting corrupted boxes and reinitialising fresh.
6. `DiaryService` is an abstract interface. `DiaryServiceHive` is the sole concrete implementation in this phase. `DiaryServiceHive` accepts its `Box<DailyEntry>` via constructor injection.
7. `saveEntry` upserts by date. Two saves for the same date result in one entry (the latest). Verified by unit test.
8. `getAllEntries` returns entries ordered by date descending.
9. `getEntryForDate` returns `null` when no entry exists for the given date. Verified by unit test.
10. `SymptomService.getPredefinedSymptoms()` returns a list (stub `[]` is acceptable). `SymptomServiceHive` accepts its `Box<Symptom>` via constructor injection.
11. `PresetService` interface and `PresetServiceHive(Box<Preset>)` implementation exist.
12. Stub `AsyncNotifierProvider` exists for both `DiaryService` and `SymptomService`.
13. Hive adapter round-trip tests exist for all `@HiveType` models (`DailyEntry`, `Symptom`, `Preset`).
14. All `DiaryService`, `SymptomService`, and `PresetService` methods have at least one unit test.
15. `deleteCustomSymptom` with a non-existent ID completes without throwing. Verified by unit test.
16. `flutter analyze` passes with zero issues.
17. `flutter test` passes with all tests green.
18. `flutter build apk --debug` succeeds.

---

## Risks and Mitigations

| Risk                                                                              | Likelihood | Impact | Mitigation                                                                                          |
| --------------------------------------------------------------------------------- | ---------- | ------ | --------------------------------------------------------------------------------------------------- |
| `flutter_secure_storage` requires Android `minSdkVersion >= 18`                  | Low        | Low    | Phase 1 sets minSdk 23; already satisfied.                                                         |
| `flutter_secure_storage` Android manifest configuration required for API 31      | Low        | Medium | Stage 4 pre-step: verify package's Android setup docs at pinned version before running pub get.    |
| `build_runner` version conflict with Phase 1 dev dependencies                    | Low        | Low    | Add `hive_generator` and `build_runner` together in Stage 1; resolve immediately.                  |
| Phase 1 directory structure not yet in place                                      | Medium     | Medium | Stage 1 should verify Phase 1 paths exist before creating model files.                             |
| Test teardown leaving temp Hive files between test runs                           | Low        | Low    | Use `Hive.deleteFromDisk()` in `tearDown`; confirmed pattern in Assumptions.                       |
| Key mismatch on uninstall/reinstall causes crash                                  | Low        | High   | Stage 4: `EncryptedStorage.init()` catches decrypt exception, deletes corrupted boxes, starts fresh. |

---

## Handoff Target

**df-qa** — review for: model design completeness, typeId reservation strategy, OQ1 resolution, `EncryptedStorage` design, test approach viability, acceptance criteria coverage, and dependency version choices.
