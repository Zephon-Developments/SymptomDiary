# SymptomDiary Coding Standards

## Purpose

These standards define the default engineering expectations for SymptomDiary.

## Core Principles

- Simplicity first.
- Reliability over cleverness.
- Privacy by default.
- Testable logic over tightly coupled UI code.
- Fast iteration without abandoning maintainability.

## Product Constraints

- Android-first Flutter application.
- Single-user and local-first for initial releases.
- Focus on symptom capture, pain tracking, quick presets, and daily freeform notes.
- Avoid multi-profile, backend, and sync complexity until explicitly required.

## Architecture Defaults

- Keep a clear separation between UI, state/domain logic, and persistence.
- Use explicit typed models rather than loosely shaped maps outside persistence boundaries.
- Prefer small services for persistence and business rules.
- Keep widgets readable; extract private widgets when a screen becomes dense.

## State Management

- Riverpod (`flutter_riverpod`) is the selected state management solution.
- Organise providers alongside the features they serve (e.g., `lib/features/diary/diary_provider.dart`).
- Prefer `AsyncNotifierProvider` for async data sources (e.g., diary entries loaded from storage).
- Prefer `NotifierProvider` for synchronous state.
- Avoid global mutable state outside Riverpod providers.

## Platform Targets

- Target platform: Android only (v1).
- Minimum Android SDK: API 23 (Android 6.0). Required by `local_auth` for biometric support; practical floor for all v1 package dependencies.
- Target and maximum Android SDK: API 31 (Android 12). Android 13 and above is not in scope for v1.
- compileSdkVersion: 31.
- Do not introduce dependencies that require API 32 or above without explicit approval.

## Dart and Flutter Conventions

- Types, enums, and extensions use PascalCase.
- Methods, variables, and properties use camelCase.
- Private members use `_camelCase`.
- Files and directories use snake_case.
- Prefer `const` constructors and widgets where possible.
- Keep imports grouped and ordered: SDK, Flutter, packages, project.

## Validation and Quality Gates

- Run `dart format` on changed Dart files.
- Run `flutter analyze` before finishing a change.
- Run the narrowest relevant `flutter test` scope for changed behavior.
- For release-impacting changes, verify a successful Android build.
- Call out any unrun checks in the final handoff.

## Documentation Obligations

- Update plans and handoffs when stage state changes.
- Update reference docs when architecture, workflow, or conventions change.
- Keep implementation summaries for completed work under `Documentation/Records/Implementation-Summaries/`.

## Workflow Rule

- All workflow documentation must stay inside this repository under `Documentation/`.
