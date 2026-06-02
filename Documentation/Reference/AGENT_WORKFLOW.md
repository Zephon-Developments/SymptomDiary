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

## Routing Resolution Rules

- Read `## Current Stage and Status` before acting on any handoff.
- Treat the `Active routing` line as the authoritative next-step signal.
- If a handoff is marked `Closed`, it is not actionable for the addressed role even if older findings or a stale requested next action remain elsewhere in the file.
- If no handoff is active for the current role, stop and redirect to the role named in the current active routing.

## Required Handoff Contents

- Summary
- Current stage and status
- Active routing
- Scope and constraints
- Files changed or expected to change
- Validation run and results
- Open blockers or risks
- Explicit next action

## Closed-Handoff Hygiene

- When a handoff is closed, update its summary and explicit next action so they match the closed state.
- Move older findings into a clearly resolved-history section or remove them if they are no longer needed.
- Do not leave stale review requests or blocking language in a closed handoff.

## Repository Boundary Rule

- Plans and handoffs must remain inside this repository.
- Do not place workflow artifacts in profile-level folders or outside the repo root.
