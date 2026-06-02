# SymptomDiary Handoff Documents

## Purpose

Handoff files preserve the latest workflow state between agents inside the repository.

## Location

- Store handoffs in `Documentation/Handoffs/`.

## Latest-Handoff Rule

- Keep one latest handoff file per source-target pair.
- Overwrite the existing handoff rather than creating dated copies unless explicitly requested.

## Naming Convention

- Use `<source>_to_<target>.md`.

Examples:

- `df-coordinator_to_df-planner.md`
- `df-planner_to_df-qa.md`
- `df-qa_to_df-coder.md`
- `df-coder_to_df-qa.md`
- `df-qa_to_df-coordinator.md`

## Required Handoff Sections

- Summary
- Current stage and status
- Context and constraints
- Files changed or planned
- Validation results
- Open blockers or risks
- Requested next action

## Boundary Rule

- Do not place handoff files outside this repository.
