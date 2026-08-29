# TASK-01-audit-vision-discovery-and-first-slice-spec

- Phase: PHASE-00-product-context-audit
- Horizon: DISCOVERY
- Specification state: DEFINED
- Execution state: READY
- Objective: In one pass, read every canonical and first-slice source and produce a single context audit the rest of DISCOVERY can consume.
- In scope: One artifact `.cursor/docs/product-discovery/00-product-context-audit.md` covering: product name (PETIA), owner (HYPERION), dual mission, pet-first, Command OS, what is stub vs contract, first-slice loop, command catalog summary.
- Out of scope: Code, stack, filling `VISION.md` unless this Task is later authorized to edit canonical docs, executing the implementation plan.
- Hard dependencies: none
- Soft dependencies: none
- Synchronization: none
- Allowed files: `.cursor/docs/product-discovery/00-product-context-audit.md`, `.cursor/docs/product-discovery/assumptions.md`
- Forbidden: any application source, modifying protected control files, modifying the first-slice spec
- Acceptance criteria:
  - Audit states which root docs are stubs (`ARCHITECTURE.md`, empty `README.md`, thin `PRINCIPLES.md`) vs which are contracts (first-slice spec, Goal).
  - Implementation plan is labeled proposal, not Accepted.
  - Feature discovery is labeled universe, not MVP backlog.
- Evidence: file exists; inspection quotes paths and stub vs contract for each source.
