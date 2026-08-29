# TASK-01-audit-repository-reality

- Phase: PHASE-07-repository-audit
- Horizon: BOOTSTRAP
- Specification state: DEFINED
- Execution state: READY
- Objective: Inventory the workspace as it is. One lot.
- In scope: `.cursor/docs/product-discovery/07-repository-audit.md`
- Out of scope: Creating `src/`, package manifests, inventing modules, copying Bellia.
- Hard dependencies: none
- Soft dependencies: none
- Synchronization: none
- Allowed files: `.cursor/docs/product-discovery/07-repository-audit.md`
- Forbidden: product application code; fake test pipelines
- Acceptance criteria:
  - States NOT PRESENT vs PRESENT for: application source, package manifest, tests, CI, phases/, tasks/, first-slice spec, implementation plan, canonical docs.
  - Does not assume files that are not on disk.
- Evidence: audit produced by listing the workspace, not memory.
