# TASK-01-prove-orchestrator-can-select-from-disk

- Phase: PHASE-09-execution-system
- Horizon: BOOTSTRAP
- Specification state: DEFINED
- Execution state: READY
- Objective: Prove cold discovery: Goal → phases on disk → tasks on disk → one eligible Task. Record the proof in one artifact. Do not start PHASE-14.
- In scope: `.cursor/docs/product-discovery/09-execution-loop-proof.md` plus a cycle-output-shaped section naming the next eligible Task.
- Out of scope: Executing DISCOVERY or MVP product work in this Task beyond writing the proof; modifying protocols.
- Hard dependencies: PHASE-07 `DONE`, PHASE-08 `DONE`
- Soft dependencies: instantiated Phase/Task tree
- Synchronization: none
- Allowed files: `.cursor/docs/product-discovery/09-execution-loop-proof.md`
- Forbidden: product code; selecting POST-MVP; marking other Tasks DONE
- Acceptance criteria:
  - Proof names actual filenames on disk.
  - Next eligible Task is PHASE-00 TASK-01 if DISCOVERY not started, or the true next unit per `ORCHESTRATOR.md` after inspecting execution states (all start READY).
  - States that BOOTSTRAP and DISCOVERY spines may proceed in parallel.
- Evidence: listed `phases/PHASE-*.md` and first Task path.
