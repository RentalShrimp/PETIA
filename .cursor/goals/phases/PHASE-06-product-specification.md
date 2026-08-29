# PHASE-06-product-specification

- Horizon: DISCOVERY
- Specification state: DEFINED
- Execution state: READY
- Objective: Produce the canonical product specification that later contract Phases consume, traced to the first-slice spec already on disk.
- In scope: Specification artifact under `.cursor/docs/product-discovery/`. Sufficient product definition per `DECISION_AUTONOMY.md` §6.
- Out of scope: Code, stack, treating this Phase `DONE` as PHASE-14 authorization.
- Hard dependencies: PHASE-00, PHASE-01, PHASE-02, PHASE-03, PHASE-04, PHASE-05 all `DONE`
- Soft dependencies: `.cursor/docs/superpowers/specs/2026-08-29-petia-command-os-design.md`
- Synchronization: none
- Blocking: none
- Tasks:
  - `TASK-01-write-canonical-product-specification`
  - `TASK-02-trace-specification-to-first-slice-spec`
- Acceptance criteria:
  - Spec can feed PHASE-10…12 without a new massive discovery round.
  - Traceability to the first-slice spec is explicit; contradictions are listed, not silently overwritten.
  - No product code.
  - DISCOVERY `DONE` is not code authorization.
