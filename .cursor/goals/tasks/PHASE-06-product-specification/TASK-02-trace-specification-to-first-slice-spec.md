# TASK-02-trace-specification-to-first-slice-spec

- Phase: PHASE-06-product-specification
- Horizon: DISCOVERY
- Specification state: DEFINED
- Execution state: READY
- Objective: Trace PHASE-06 specification sections to `.cursor/docs/superpowers/specs/2026-08-29-petia-command-os-design.md`. One lot. List contradictions; do not silently overwrite the first-slice spec.
- In scope: `.cursor/docs/product-discovery/06-traceability.md`
- Out of scope: Editing the first-slice spec (protected), code.
- Hard dependencies: TASK-01 of this Phase `DONE`
- Soft dependencies: none
- Synchronization: none
- Allowed files: `.cursor/docs/product-discovery/06-traceability.md`, `.cursor/docs/product-discovery/assumptions.md`
- Forbidden: modifying the first-slice spec; product code
- Acceptance criteria:
  - Table: spec section → discovery artifact → status (aligned / gap / contradiction).
  - Contradictions are BLOCKING notes for Human, not auto-resolved against Goal.
- Evidence: traceability table complete for spec sections 1–14 relevant to the wave.
