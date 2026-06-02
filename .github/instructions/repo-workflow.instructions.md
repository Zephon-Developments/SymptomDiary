---
description: "Use when working in SymptomDiary. Ensures plans, handoffs, and workflow documentation stay inside this repository under Documentation/."
name: "SymptomDiary Repo Workflow"
applyTo: "**"
---
# SymptomDiary Workflow Location Policy

- All workflow artifacts must be created inside this repository.
- Use repo-relative paths only.
- Never create plan or handoff files in user-profile folders or outside the repository root.

# Required Locations

- Plans: `Documentation/Plans/`
- Handoffs: `Documentation/Handoffs/`
- Standards and workflow references: `Documentation/Reference/`
- Completed implementation summaries: `Documentation/Records/Implementation-Summaries/`
- Superseded workflow artifacts: `Documentation/Archive/`

# Latest-State Policy

- Keep the active plan updated in place.
- Keep the latest handoff for each source-target direction updated in place.
- Do not create dated handoff variants unless explicitly requested.

# Routing Resolution Policy

- Before acting on a handoff, read `## Current Stage and Status` first.
- The `Active routing` line is the source of truth for who acts next.
- A handoff marked `Closed` is historical only. Do not continue an older requested next action, findings list, or review focus from a closed handoff.
- If no handoff is active for the current role, stop and report the currently active route instead of reopening a finished stage.

# Closed-Handoff Hygiene

- When closing a handoff, rewrite its `Summary`, `Requested Next Action`, and any review-focus text so they agree with the closed state.
- Keep historical findings only when they are clearly marked resolved or archived.
- Do not leave stale blocker language or role-routing instructions in a closed handoff.

# Stage Tracking Policy

- Every non-trivial task must have a plan with a progress sheet.
- Every stage transition must be reflected in the plan and in the latest written handoff.
- Work should be resumable from the current repo state without relying on chat history.