# Copilot Instructions for SymptomDiary

## Project Overview
SymptomDiary is an Android-first Flutter application for patients with long-term conditions to record symptoms, pain levels, quick preset reports, and freeform daily diary entries.

## Product Direction
- Prioritize rapid delivery and low complexity.
- Keep the first release local-first and offline-first.
- Optimize for fast symptom capture and clear historical review.
- Favor simple single-user flows over multi-profile or backend-heavy architecture.

## Workflow and Documentation Rules
- All planning, handoff, and workflow documentation must stay inside this repository.
- Store plans in `Documentation/Plans/`.
- Store handoffs in `Documentation/Handoffs/`.
- Store standards and workflow references in `Documentation/Reference/`.
- Store completed implementation summaries in `Documentation/Records/Implementation-Summaries/`.
- Store superseded workflow artifacts in `Documentation/Archive/`.
- Never create plan or handoff files under profile folders, workspace-level folders, or outside this repository root.

## Latest-State Rule
- Keep the latest workflow state easy to resume.
- Prefer updating the active plan file rather than creating unnecessary new variants.
- Handoffs should keep one latest file per source-target pair unless the user explicitly asks for historical snapshots.

## Engineering Defaults
- Use Flutter/Dart conventions and keep architecture simple.
- Prefer explicit models, predictable state, and small testable services.
- Validate changes with the narrowest relevant checks first.
- Update workflow docs when stage status, scope, or decisions change.

## Current Build Context
- This repository is currently in bootstrap/planning state.
- If project code is scaffolded later, add concrete build, analysis, and test commands to `Documentation/Reference/CODING_STANDARDS.md`.