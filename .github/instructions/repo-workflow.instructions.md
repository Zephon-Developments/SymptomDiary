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

# Stage Tracking Policy

- Every non-trivial task must have a plan with a progress sheet.
- Every stage transition must be reflected in the plan and in the latest written handoff.
- Work should be resumable from the current repo state without relying on chat history.