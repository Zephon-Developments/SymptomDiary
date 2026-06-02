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
- Active routing
- Context and constraints
- Files changed or planned
- Validation results
- Open blockers or risks
- Requested next action

## Routing Precedence

- Read `Current stage and status` before any other section.
- Use the `Active routing` line as the source of truth for the next actor.
- Treat closed handoffs as historical context only.

## Closing a Handoff

- When a handoff is closed, rewrite the summary and requested next action so they match the closed state.
- Remove or clearly archive stale findings, review requests, and blocker language.
- Do not leave a closed handoff pointing the next action back to the role that has already finished.

## Boundary Rule

- Do not place handoff files outside this repository.
