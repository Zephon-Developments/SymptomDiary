# Plan: Phase 6 — Printable Reports

**Created:** 2026-06-02  
**Status:** Approved — Ready for Implementation (after Phase 5) — QA cycle 1 approved 2026-06-02  
**Parent Plan:** `Documentation/Plans/Plan_Initial_App_Delivery.md`  
**Prerequisite:** Phase 5 complete and approved by df-qa.  
**Active Phase:** Phase 6 — Printable Reports

---

## Objective

Replace the Phase 1 `ReportScreen` placeholder with a working PDF report generator. The user selects a date range, taps Generate, and receives a PDF of their symptom history via the Android share sheet. The PDF is constructed in memory; no persistent unencrypted file is written to disk.

---

## Constraints

- Read all entry data through `DiaryService` (Phase 2). No direct Hive calls in UI or service code.
- Read symptom names through `SymptomsNotifier` (Phase 4). No direct Hive calls for symptom resolution.
- PDF must be generated in memory (`Uint8List`) and shared via Android share sheet. No persistent plaintext file.
- The `printing` package writes a transient file to the app-private cache directory to satisfy the Android share intent — this is acceptable (private to app, not persisted, not user-accessible storage).
- Android-first. iOS not in scope for v1.
- Do not introduce dependencies that require API 32 or above.
- Rapid delivery and low complexity are product priorities. Prefer a simple readable PDF layout over elaborate design.

---

## Assumptions

1. Phases 1–5 are complete. `DiaryServiceHive.getAllEntries()` (Phase 5), `SymptomsNotifier` (Phase 4), and the `ReportScreen` placeholder (Phase 1) are all available.
2. `pdf: ^3.10.0` and `printing: ^5.12.0` are the chosen packages (see OQ1). Both are pure-Dart/Flutter with no API 32+ requirements and are compatible with minSdkVersion 23 / compileSdkVersion 31.
3. `ReportService.generatePdf()` is synchronous — the `pdf` package constructs documents without async I/O. `Printing.sharePdf()` is async (invokes the share intent).
4. Entries are fetched on demand (one-shot `await service.getAllEntries()`) when the user taps Generate — not reactively watched. This avoids coupling the Reports feature to the History feature's `historyProvider`.
5. Symptom names are fetched via `await ref.read(symptomsProvider.future)` at generation time.
6. Date range filter logic: include entries where `date >= range.start && date <= range.end`, using normalised midnight dates (Phase 2 contract).
7. An empty result (no entries in range) shows a message and does not trigger PDF generation or the share sheet.

---

## Open Questions

| ID  | Question | Status |
| --- | -------- | ------ |
| OQ1 | `pdf ^3.10.0 + printing ^5.12.0` vs alternative (e.g. `syncfusion_flutter_pdf`)? Default: `pdf + printing` — established Dart-native stack, no licensing cost, no API 32+ dependency. | Open — coordinator/owner input before Stage 1; default applied. |
| OQ2 | Report layout: one section per day (header + pain + symptoms + notes) vs a compact table (one row per day)? Default: one section per entry — more readable for notes-heavy entries. | Open — low risk; default applied. |

---

## Package Dependencies

Two new dependencies:

```yaml
dependencies:
  pdf: ^3.10.0
  printing: ^5.12.0
```

All other dependencies already present from Phases 2–5.

---

## Directory Structure (additions to Phase 5 scaffold)

```
lib/
  features/
    reports/
      screens/
        report_screen.dart          # Replaces Phase 1 placeholder — date range config + generate
      services/
        report_service.dart         # Pure PDF generation logic (no Hive, no Riverpod)
      providers/
        report_provider.dart        # Provider<ReportService> — simple factory
test/
  features/
    reports/
      report_service_test.dart
```

---

## Data Model Additions

None. No new Hive models or state classes.

---

## Stage Plan

### Stage 1 — Add Dependencies

Add to `pubspec.yaml` and run `flutter pub get`:

```yaml
dependencies:
  pdf: ^3.10.0
  printing: ^5.12.0
```

Verify `flutter analyze` remains clean after package resolution.

**Owner:** df-coder  
**Completion Criteria:** `pdf` and `printing` resolve without conflicts. `flutter analyze` passes. OQ1 status noted.

---

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

- Method is async — calls `await doc.save()` internally (`pdf` package `Document.save()` returns `Future<Uint8List>`). No other I/O; no Hive dependency; no Riverpod dependency.
- PDF structure:
  - **Header**: "Symptom Diary Report", date range formatted as "DD MMM YYYY – DD MMM YYYY".
  - **Entry sections** (one per entry, sorted by date ascending):
    - Date line: formatted full date (e.g. "Monday, 2 June 2026").
    - Pain level: "Pain: N/10".
    - Symptoms: "Symptoms: name1, name2, …" — names resolved by filtering `symptoms` where `symptom.id` is in `entry.symptomIds`. If no symptoms: "Symptoms: None".
    - Notes: rendered as a paragraph. If empty: omit the notes line.
  - **Footer**: "Generated: [generation date]".
- `symptoms` list is passed in — no Hive dependency in this class.
- `reportServiceProvider = Provider<ReportService>((ref) => ReportService())` in `report_provider.dart`.

**Owner:** df-coder  
**Completion Criteria:** `ReportService.generatePdf()` returns a non-empty `Future<Uint8List>`. Structure includes header, at least one entry section, and footer. No Hive or Riverpod imports in `report_service.dart`.

---

### Stage 3 — Report Screen

Replace `ReportScreen` Phase 1 placeholder at `lib/features/reports/screens/report_screen.dart`:

- `ReportScreen` is a `ConsumerStatefulWidget` with local `_isLoading`, `_dateRange`, and `_errorMessage` state fields.
- UI elements:
  - **Date range row**: `TextButton` or `OutlinedButton` labelled with the current range (or "Select date range" if none selected). Tapping calls `showDateRangePicker`. `lastDate` restricted to today (`DateTime.now()` normalised to midnight). `firstDate`: `DateTime(2000)` (safe floor).
  - **"Generate Report" button**: disabled when `_dateRange` is null or `_isLoading` is true. Shows `CircularProgressIndicator` during generation.
  - **Error message**: shown below the button if `_errorMessage` is non-null.
- Generate flow (called on button tap):

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

- No `/report` named route change needed — `ReportScreen` is already registered via the Phase 1 navigation shell's Reports tab.

**Owner:** df-coder  
**Completion Criteria:** Date range picker opens and stores selection. Generate button triggers flow. Loading indicator shown during generation. Empty range shows message without triggering share. Share sheet opens on success with a non-empty PDF. Error state shows on failure.

---

### Stage 4 — Tests

Tests covering:

- `ReportService.generatePdf()`: single entry → non-empty bytes; multiple entries → bytes larger than single entry; empty entries list (guard against accidental call) → still returns non-empty PDF (empty-state is handled in screen, not service).
- `ReportService.generatePdf()`: symptom names resolved correctly — entry with known `symptomIds` — assert the call completes without throwing and returns non-empty bytes. Do **not** assert symptom name literals in raw `Uint8List` bytes: PDF content streams encode text via font tables and string presence in raw bytes is unreliable across `pdf` package versions.
- `ReportService.generatePdf()`: entry with no symptoms → no crash; entry with no notes → no crash.
- Widget test for `ReportScreen`: Generate button disabled with no date range. Shows loading state when generating (pump and check). Shows "No entries found" message when `diaryServiceProvider` returns empty list for range.

All provider tests use `ProviderScope(overrides: [...])` with mock services.

**Owner:** df-coder  
**Completion Criteria:** `flutter test` passes. `ReportService` unit tests cover populated, empty-symptoms, and empty-notes cases. Widget test covers disabled state and empty result state.

---

### Stage 5 — Validation

1. `dart format lib/ test/`
2. `flutter analyze` — zero issues
3. `flutter test` — all tests pass
4. `flutter build apk --debug` — succeeds
5. Manual smoke on emulator: Reports tab → tap "Select date range" → pick range that includes entries → tap "Generate Report" → loading indicator shows → Android share sheet opens → open PDF → verify header, entry sections, and footer are present and readable. Then: select range with no entries → verify "No entries found" message shown, share sheet not triggered.

**Owner:** df-coder  
**Completion Criteria:** All five checks pass. OQ1 status noted in handoff.

---

## Progress Sheet

| Stage | Owner | Status | Completion Criteria | Notes |
| --- | --- | --- | --- | --- |
| 1 — Add Dependencies | df-coder | Not Started | pdf + printing resolve; flutter analyze passes | OQ1 open — confirm package choice |
| 2 — Report Service | df-coder | Not Started | generatePdf() produces non-empty Uint8List; no Hive or Riverpod in service | Pure class; Provider<ReportService> factory |
| 3 — Report Screen | df-coder | Not Started | Date range picker works; generate triggers flow; loading/error/empty states correct; share sheet opens | ConsumerStatefulWidget; local state only |
| 4 — Tests | df-coder | Not Started | flutter test passes; ReportService unit tests; widget tests cover disabled and empty-result states | ProviderScope overrides with mock services |
| 5 — Validation | df-coder | Not Started | format/analyze/test/build/smoke all pass | Note OQ1 status in handoff |

---

## Acceptance Criteria

1. Reports tab shows a date range selection button and a "Generate Report" button.
2. "Generate Report" button is disabled when no date range is selected.
3. Tapping the date range button opens `showDateRangePicker`; `lastDate` is today; selection is displayed on the button.
4. Tapping "Generate Report" shows a loading indicator while the PDF is being generated and shared.
5. PDF is generated in memory as `Uint8List`; no persistent unencrypted file is written to app storage.
6. Generated PDF includes a title, the selected date range, and a section per entry (date, pain level, symptom names, notes).
7. Symptom names in the PDF are resolved from `symptomsProvider` (predefined + custom, by ID lookup).
8. Entries in the PDF are sorted by date ascending.
9. Android share sheet opens on successful generation; the user can select a share target (e.g. email, drive, files).
10. If no entries exist in the selected date range, a "No entries found" message is shown and the share sheet is not triggered.
11. If PDF generation throws an unexpected error, an error message is shown and the share sheet is not triggered.
12. Generating a PDF does not require API level 32 or above.
13. `ReportService.generatePdf()` is an asynchronous function returning `Future<Uint8List>` — no Hive dependency, no Riverpod dependency.
14. `ReportService` unit tests cover: populated entries, entry with no symptoms, entry with no notes.
15. `ReportScreen` widget tests cover: disabled Generate button with no date range; "No entries found" message when service returns empty list.
16. `flutter analyze` passes with zero issues.
17. `flutter test` passes with all tests green.
18. `flutter build apk --debug` succeeds.
19. Manual smoke test on emulator confirms: date range selection, successful PDF generation, share sheet open, PDF content readable, empty-range message shown.

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| `pdf` package version incompatible with compileSdkVersion 31 | Low | Low | Package is pure Dart — no native Android code; no SDK constraint |
| `printing` package `Printing.sharePdf()` writes unencrypted temp file to app cache | High | Low | Acceptable per policy: file is in app-private cache, transient, not persistent storage |
| OQ1 (`pdf + printing` vs alternative) unresolved | High | Low | Default packages applied; only the service implementation changes if OQ1 resolves differently |
| Large date range (e.g. 2 years) produces very large in-memory PDF | Low | Low | In-memory PDF for 730 entries is well under typical Android memory limits; no pagination required for v1 |
| `symptomsProvider.future` rejects (e.g. Hive error) during generation | Low | Medium | Caught by the try/catch in `_generate`; user sees error message |
| `getAllEntries()` not yet added to `DiaryService` in Phase 5 | Low | Low | Phase 5 Stage 1 adds it; this plan assumes Phase 5 complete |

---

## Handoff Target

**df-qa** — cycle 1 re-review. All M1 findings addressed (async signature, await call site, AC13). L1 implementation note applied to Stage 4 test description. No prior findings remain open.
