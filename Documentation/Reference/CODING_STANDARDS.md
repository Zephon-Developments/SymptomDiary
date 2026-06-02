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
