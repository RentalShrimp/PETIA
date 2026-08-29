# PHASE-09-execution-system

- Horizon: BOOTSTRAP
- Specification state: DEFINED
- Execution state: READY
- Objective: Prove the execution loop can discover Goal, Phases, and Tasks from disk and select one eligible unit without inventing work.
- In scope: A cycle-output demonstration (in session) that the instantiated tree is selectable. Optional short note under `.cursor/docs/product-discovery/execution-loop.md`.
- Out of scope: Implementing PHASE-14, modifying protected protocols except if Human authorized, creating a state database.
- Hard dependencies: PHASE-07 `DONE`, PHASE-08 `DONE`
- Soft dependencies: none
- Synchronization: none
- Blocking: none
- Tasks:
  - `TASK-01-prove-orchestrator-can-select-from-disk`
- Acceptance criteria:
  - A cold agent can name the next eligible Task from disk + `ORCHESTRATOR.md`.
  - Horizon gate DISCOVERY ∪ BOOTSTRAP ∪ MVP is respected; POST-MVP not selected.
  - `IMPLEMENT → DONE` remains unreachable; this Phase does not ship product.
