# Plan: Phase 5 — History Review

**Created:** 2026-06-02  
**Status:** Approved — Ready for Implementation (after Phase 4)  
**Parent Plan:** `Documentation/Plans/Plan_Initial_App_Delivery.md`  
**Prerequisite:** Phase 4 complete and approved by df-qa.  
**Active Phase:** Phase 5 — History Review

---

## Objective

Replace the Phase 1 `HistoryScreen` placeholder with a navigable, filterable view of past diary entries. The user can see a monthly calendar of entries with visual pain level and symptom indicators, drill into entry details, and filter history by date range, symptom, or pain level threshold.

---

## Constraints

- Read all data through `DiaryService` and `SymptomService` (Phase 2). No direct Hive calls in UI or provider code.
- Use existing `DiaryServiceHive` and `SymptomsNotifier` (Phase 4) — no new persistence layer.
- Rapid delivery and low complexity are product priorities (CODING_STANDARDS). Prefer simple, working UI over elaborate visualisations.
- Android-first. iOS not in scope for v1.
- Do not introduce dependencies that require API 32 or above.

---

## Assumptions

1. Phase 4 `DiaryNotifier` (family provider keyed by `DateTime`) and `SymptomsNotifier` (merged predefined + custom list) are complete and available.
2. Phase 2 `DiaryService.getAllEntries()` (or equivalent) returns all `DailyEntry` objects from Hive. If this method does not yet exist, df-coder adds it to `DiaryServiceHive` during Stage 1.
3. The History tab is already registered in the Phase 1 navigation shell (`HistoryScreen` placeholder exists at `lib/features/history/screens/history_screen.dart`).
4. `DailyEntry.date` is always normalised to midnight local (Phase 2 contract). Calendar keys derive from this field.
5. `table_calendar ^3.1.0` is used for the calendar widget (see OQ1). It is compatible with minSdkVersion 23 and compileSdkVersion 31.
6. Filter state is synchronous (`NotifierProvider`) — it holds the active filter criteria and derives the filtered entry list from the loaded `HistoryNotifier` state.
7. Tapping a day with an existing entry opens a read-only `EntryDetailScreen`. Tapping an empty past day navigates to `LogEntryScreen` for that date (reuses Phase 4 screen).
8. Entry detail is read-only in this phase. Edit navigates back to `LogEntryScreen`.

---

## Open Questions

| ID  | Question | Status |
| --- | -------- | ------ |
| OQ1 | Use `table_calendar` package or a custom simple month grid? Default: `table_calendar ^3.1.0` — well-established, handles month navigation and marker builders, low custom code burden. | Open — coordinator/owner input before Stage 2; placeholder implementation uses `table_calendar`. |
| OQ2 | Should tapping an empty future day be blocked, or should the date picker guard be enough? Default: show no tap action on future days (same restriction as Phase 4 date picker). | Open — low risk; default applied. |

---

## Package Dependencies

One new dependency:

```yaml
dependencies:
  table_calendar: ^3.1.0
```

All other dependencies already present from Phases 2–4 (`hive`, `hive_flutter`, `flutter_riverpod`, `uuid`).

---

## Directory Structure (additions to Phase 4 scaffold)

```
lib/
  features/
    history/
      screens/
        history_screen.dart          # Replaces Phase 1 placeholder — monthly calendar + filter bar
        entry_detail_screen.dart     # Read-only entry detail view
      providers/
        history_provider.dart        # AsyncNotifierProvider<HistoryNotifier, List<DailyEntry>>
        history_filter_provider.dart # NotifierProvider<HistoryFilterNotifier, HistoryFilterState>
      widgets/
        entry_calendar.dart          # table_calendar wrapper with pain-level event markers
        entry_day_marker.dart        # Single-day marker: pain colour dot + symptom presence dot
        filter_sheet.dart            # Bottom sheet: date range, symptom multi-select, pain threshold
test/
  features/
    history/
      history_provider_test.dart
      history_filter_provider_test.dart
```

---

## Data Model Additions

No new Hive models. One new immutable state class (no Hive annotation):

```dart
class HistoryFilterState {
  final DateTimeRange? dateRange;       // null = no date filter
  final List<String> symptomIds;        // empty = no symptom filter
  final int? painThreshold;             // null = no threshold; show entries >= threshold
  const HistoryFilterState({
    this.dateRange,
    this.symptomIds = const [],
    this.painThreshold,
  });
  HistoryFilterState copyWith({...});
}
```

---

## Stage Plan

### Stage 1 — Wire History Provider

Add `HistoryNotifier` loading all diary entries from Hive via `DiaryService`:

- Provider type: `AsyncNotifierProvider<HistoryNotifier, List<DailyEntry>>`
- `build()` calls `service.getAllEntries()`. If `getAllEntries()` does not exist on `DiaryServiceHive`, add it: iterate `box.values.toList()` sorted by date descending.
- No family key — loads the full history in one provider; individual-entry access still uses `diaryProvider(date)` from Phase 4.
- `HistoryFilterNotifier` (`NotifierProvider<HistoryFilterNotifier, HistoryFilterState>`): holds active filter criteria. `build()` returns `const HistoryFilterState()` (no filters). Methods: `setDateRange(DateTimeRange?)`, `setSymptomFilter(List<String>)`, `setPainThreshold(int?)`, `clearAll()`.
- Derived filtered list: computed inside `HistoryScreen` widget by watching both providers:

  ```dart
  final allEntries = ref.watch(historyProvider).valueOrNull ?? [];
  final filter = ref.watch(historyFilterProvider);
  final filtered = _applyFilter(allEntries, filter);
  ```

  `_applyFilter` is a pure function — no provider involved.

**Owner:** df-coder  
**Completion Criteria:** `HistoryNotifier.build()` loads all entries from Hive. `HistoryFilterNotifier` holds filter state and exposes mutation methods. `DiaryServiceHive.getAllEntries()` exists and returns all entries sorted by date descending. State can be invalidated via `ref.invalidate(historyProvider)` to reload all entries from Hive.

---

### Stage 2 — History Screen — Calendar View

Replace `HistoryScreen` placeholder:

- Add `table_calendar: ^3.1.0` to `pubspec.yaml` and run `flutter pub get`.
- `TableCalendar` widget configured with:
  - `firstDay`: earliest entry date (or a fixed floor e.g. 2 years ago if no entries).
  - `lastDay`: today (`DateTime.now()` normalised to midnight).
  - `focusedDay`: today on first load.
  - `calendarFormat`: month (default, no format toggle needed).
  - `eventLoader`: returns `List<DailyEntry>` for a given day from the filtered entries map (pre-build a `Map<DateTime, DailyEntry>` keyed by normalised date).
  - `calendarBuilders.markerBuilder`: renders `EntryDayMarker` for days with an entry.
- `EntryDayMarker` widget: a small coloured dot indicating pain level (e.g. green → amber → red mapped from 0–10 in thirds), plus a second dot if any symptoms are recorded.
- Tapping a day with an entry: `Navigator.push` to `EntryDetailScreen(date: day)`.
- Tapping an empty past day (≤ today): `Navigator.push` to `LogEntryScreen(date: day)`.
- Tapping a future day: no action (OQ2 default).
- Filter FAB or icon button in app bar opens `FilterSheet` bottom sheet.
- `HistoryScreen` wraps the calendar view in a `RefreshIndicator`; `onRefresh` calls `ref.invalidate(historyProvider)` to reload all entries from Hive (ensures any entries added or edited via `LogEntryScreen` are reflected without requiring navigation restart).

**Owner:** df-coder  
**Completion Criteria:** Calendar renders current month; days with entries show `EntryDayMarker`; month navigation works; tapping entry day navigates to detail; tapping empty past day navigates to log entry screen. Pull-to-refresh reloads entries from Hive and updates calendar markers.

---

### Stage 3 — Entry Detail Screen

New read-only screen at `lib/features/history/screens/entry_detail_screen.dart`:

- Accepts `DateTime date` parameter.
- Watches `diaryProvider(date)` (Phase 4 family provider) — renders entry fields.
- Displays: formatted date, pain level (numeric + visual indicator), symptom chips (names resolved via `ref.watch(symptomsProvider)`), notes text.
- App bar action: "Edit" button navigates to `LogEntryScreen(date: date)`.
- If `diaryProvider(date)` returns null (entry deleted while screen open): show a "No entry for this date" message with a back button.
- Register `/entry_detail` as a named route in `app.dart`. Navigate via `Navigator.pushNamed(context, '/entry_detail', arguments: date)`.

**Owner:** df-coder  
**Completion Criteria:** Entry detail screen shows all fields correctly. Symptom names resolve from `symptomsProvider`. Edit navigates to `LogEntryScreen`. Null guard works.

---

### Stage 4 — Filter and Sort

Implement `FilterSheet` and wire filter state:

- `FilterSheet` is a `DraggableScrollableSheet` modal bottom sheet with three sections:
  1. **Date range** — `showDateRangePicker` button; shows selected range or "All dates". Calls `historyFilterNotifier.setDateRange(range)`.
  2. **Symptom filter** — multi-select `FilterChip` grid from `ref.watch(symptomsProvider)`. Calls `historyFilterNotifier.setSymptomFilter(selectedIds)`.
  3. **Pain threshold** — `Slider(min: 0, max: 10, divisions: 10)` labelled "Show entries with pain ≥ N". `onChanged` must cast the `double` slider value to `int` before calling: `historyFilterNotifier.setPainThreshold(value.toInt())`. (`divisions: 10` ensures the value always snaps to a whole number, so `toInt()` is lossless.)
  4. **Clear all** button — calls `historyFilterNotifier.clearAll()`.
- `HistoryScreen` shows an active-filter indicator (e.g. badge on filter icon) when any filter is non-default.
- `_applyFilter` logic:
  - Date range: exclude entries outside `filter.dateRange`.
  - Symptom filter: include only entries whose `symptomIds` contains at least one of `filter.symptomIds` (OR logic). If `filter.symptomIds` is empty: show all.
  - Pain threshold: exclude entries with `painLevel < filter.painThreshold`. If null: show all.
- Empty state: if filtered result is empty, show a centred message — "No entries match your filters" with a "Clear filters" text button.

**Owner:** df-coder  
**Completion Criteria:** Each filter type reduces the visible calendar entries correctly. Clearing filters restores all entries. Empty state shows when filters produce no results. Active-filter indicator visible when filters are set.

---

### Stage 5 — Tests

Tests covering:

- `HistoryNotifier`: `build()` returns all entries sorted by date descending; returns empty list when no entries.
- `HistoryNotifier`: state reloads correctly after `ref.invalidate(historyProvider)` — seed box with one entry, load provider, add a second entry to box directly, invalidate provider, assert both entries present in reloaded state.
- `DiaryServiceHive.getAllEntries()`: returns all entries; returns empty list when box is empty.
- `HistoryFilterNotifier`: `setDateRange`, `setSymptomFilter`, `setPainThreshold`, `clearAll` each produce correct state.
- `_applyFilter` pure function: date range filter, symptom filter (OR logic), pain threshold filter, combination of all three, no filters returns all.
- Widget test for `HistoryScreen`: shows empty state when no entries; shows calendar markers when entries present.
- Widget test for `EntryDetailScreen`: renders all fields; "Edit" navigates to `LogEntryScreen`; null guard renders correctly.

All provider tests use `ProviderScope(overrides: [...])` with mock services and box provider overrides.

**Owner:** df-coder  
**Completion Criteria:** `flutter test` passes; all filter logic paths covered by unit tests; widget tests cover empty and populated states; `HistoryNotifier` invalidation reload test passes.

---

### Stage 6 — Validation

1. `dart format lib/ test/`
2. `flutter analyze` — zero issues
3. `flutter test` — all tests pass
4. `flutter build apk --debug` — succeeds
5. Manual smoke on emulator: History tab → calendar renders → navigate months → tap entry day → detail screen → edit → back → tap empty past day → log entry → save → return to History → pull to refresh → verify new entry marker appears → open filter sheet → apply date range → symptom filter → pain threshold → clear filters.

**Owner:** df-coder  
**Completion Criteria:** All five checks pass. OQ1 status noted in handoff (placeholder `table_calendar` implementation used if unresolved).

---

## Progress Sheet

| Stage | Owner | Status | Completion Criteria | Notes |
| --- | --- | --- | --- | --- |
| 1 — Wire History Provider | df-coder | Not Started | HistoryNotifier loads all entries; HistoryFilterNotifier holds filter state; getAllEntries() exists on DiaryServiceHive; state invalidatable via ref.invalidate(historyProvider) | DiaryServiceHive.getAllEntries() may need to be added; sort descending by date |
| 2 — History Screen (Calendar) | df-coder | Not Started | Calendar renders with pain-level markers; month navigation works; day tap routing correct; pull-to-refresh reloads entries | table_calendar ^3.1.0 new dependency; OQ1 open; RefreshIndicator wraps calendar |
| 3 — Entry Detail Screen | df-coder | Not Started | All entry fields render; edit navigation works; null guard works | /entry_detail named route; reuses diaryProvider(date) and symptomsProvider from Phase 4 |
| 4 — Filter and Sort | df-coder | Not Started | Each filter type works; clear all works; empty state shows | _applyFilter is pure function; OR logic for symptom filter; Slider onChanged casts double to int via .toInt() |
| 5 — Tests | df-coder | Not Started | flutter test passes; filter logic unit tested; widget tests cover empty and populated states; HistoryNotifier invalidation reload test passes | ProviderScope overrides with mock services |
| 6 — Validation | df-coder | Not Started | format/analyze/test/build/smoke all pass | Note OQ1 status in handoff |

---

## Acceptance Criteria

1. History screen shows a monthly calendar view with visual pain level indicators per day.
2. Days with entries show a symptom presence indicator alongside the pain level dot.
3. Days without entries show no marker.
4. Tapping a day with an entry opens a read-only `EntryDetailScreen`.
5. Entry detail shows: formatted date, pain level (numeric), all symptom names (resolved from `symptomsProvider`), and notes.
6. Tapping an empty past day navigates to `LogEntryScreen` for that date.
7. Future days are not tappable.
8. Calendar navigates between months using the built-in `table_calendar` controls.
9. Date range filter limits the calendar to showing only entries within the range.
10. Symptom filter shows only entries whose `symptomIds` contain at least one selected symptom (OR logic).
11. Pain threshold filter shows only entries with `painLevel >= threshold`.
12. Applying no filters shows all entries.
13. Empty state ("No entries match your filters") shown with "Clear filters" action when filters produce no results.
14. Active filter indicator (badge or icon change) visible on filter button when any filter is non-default.
15. `HistoryNotifier` unit tests cover load-all, empty-box, and sort-order cases.
16. `_applyFilter` unit tests cover each filter type and combinations.
17. Widget tests cover empty state and populated calendar state.
18. `flutter analyze` passes with zero issues.
19. `flutter test` passes with all tests green.
20. `flutter build apk --debug` succeeds.
21. Manual smoke test on emulator confirms full history view, detail, filter, and edit flows.
22. Pull-to-refresh on History screen reloads entries from Hive and reflects any entries added or edited via `LogEntryScreen` in the current session.

---

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| `table_calendar` API changes between ^3.1.0 and latest | Low | Low | Pin to `^3.1.0`; check changelog if pub get resolves a newer patch |
| Calendar performance with large entry history (e.g. 2+ years daily) | Low | Low | `eventLoader` pre-builds a `Map<DateTime, DailyEntry>` once on provider load; O(1) lookup per day |
| History stale after entry save | Medium | Medium | `RefreshIndicator` on `HistoryScreen`; `onRefresh` calls `ref.invalidate(historyProvider)` to reload from Hive without cross-provider coupling |
| `DiaryServiceHive.getAllEntries()` not implemented in Phase 2 | High | Low | Stage 1 adds it if absent; trivial: `box.values.toList()` sorted by `entry.date` |
| Filter combination logic produces unexpected results | Medium | Low | Covered by unit tests for `_applyFilter` with all filter type combinations |
| OQ1 (`table_calendar` vs custom grid) unresolved | High | Low | Default `table_calendar` used; only the calendar widget changes if OQ1 resolves differently |

---

## Handoff Target

**df-qa** — cycle 1 re-review. All M1 (pull-to-refresh) and L1 (painThreshold cast) findings addressed. No prior findings remain open.
