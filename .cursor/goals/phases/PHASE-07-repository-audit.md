# PHASE-07-repository-audit

- Horizon: BOOTSTRAP
- Specification state: DEFINED
- Execution state: READY
- Objective: Record repository reality. Do not assume modules, stacks, or files that are not on disk.
- In scope: Audit artifact (cycle output and/or `.cursor/docs/product-discovery/repository-audit.md`). List what exists vs stubs vs missing.
- Out of scope: Creating application layout, inventing `src/`, choosing a stack, copying Bellia modules.
- Hard dependencies: none (BOOTSTRAP may run in parallel with DISCOVERY)
- Soft dependencies: none
- Synchronization: none
- Blocking: none
- Tasks:
  - `TASK-01-audit-repository-reality`
- Acceptance criteria:
  - Inventory states: no application `src/`, no package manifest, directional docs as stubs or partial, first-slice spec present, implementation plan present but not ratified, Phase/Task tree present after instantiation.
  - No invented modules.
  - No product code.
