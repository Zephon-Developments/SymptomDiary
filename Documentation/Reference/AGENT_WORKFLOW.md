# SymptomDiary Agent Workflow

## Purpose

This repository uses a documented workflow so work can be resumed from repository files rather than chat history.

## Default Flow

1. `df-coordinator` captures scope and initiates planning.
2. `df-planner` creates or updates the active plan in `Documentation/Plans/`.
3. `df-qa` reviews the plan and either blocks it for revision or approves implementation.
4. `df-coder` implements the approved stage.
5. `df-qa` reviews implementation and either requests rework or greenlights integration.
6. `df-coordinator` summarizes the result and handles commit workflow when requested.

## Source of Truth

- Active plan files live in `Documentation/Plans/`.
- Latest stage handoffs live in `Documentation/Handoffs/`.
- Standards and process references live in `Documentation/Reference/`.

## Plan Rules

- Every non-trivial task must have an active plan file.
- Plans must include a progress sheet with one row per stage.
- The active plan should be updated in place rather than replaced unnecessarily.

## Handoff Rules

- Every stage transition requires a written handoff file.
- Handoff files should keep one latest file per source-target pair.
- Default handoff examples:

  - `Documentation/Handoffs/df-coordinator_to_df-planner.md`
  - `Documentation/Handoffs/df-planner_to_df-qa.md`
  - `Documentation/Handoffs/df-qa_to_df-coder.md`
  - `Documentation/Handoffs/df-coder_to_df-qa.md`
  - `Documentation/Handoffs/df-qa_to_df-coordinator.md`

## Required Handoff Contents

- Summary
- Current stage and status
- Scope and constraints
- Files changed or expected to change
- Validation run and results
- Open blockers or risks
- Explicit next action

## Repository Boundary Rule

- Plans and handoffs must remain inside this repository.
- Do not place workflow artifacts in profile-level folders or outside the repo root.
