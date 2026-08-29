# TASK-01-define-execution-and-docs-layout

- Phase: PHASE-08-project-structure
- Horizon: BOOTSTRAP
- Specification state: DEFINED
- Execution state: READY
- Objective: Define where discovery artifacts, goals, and canonical docs live. Defer application layout to PHASE-13.
- In scope: `.cursor/docs/product-discovery/08-layout-rules.md`
- Out of scope: Implementing app structure, Docker, stack.
- Hard dependencies: PHASE-07 `DONE`
- Soft dependencies: none
- Synchronization: none
- Allowed files: `.cursor/docs/product-discovery/08-layout-rules.md`
- Forbidden: creating `src/` or evidence stores; extra control scripts
- Acceptance criteria:
  - Discovery artifacts: `.cursor/docs/product-discovery/`.
  - Goals: `.cursor/goals/{phases,tasks}/`.
  - No `.cursor/goals/history/`, no evidence directory.
  - Application paths marked DEFERRED until architecture ratification.
- Evidence: layout rules file exists and matches Goal constraints.
