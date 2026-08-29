# PHASE-10-mvp-product-baseline

- Horizon: MVP
- Specification state: DEFINED
- Execution state: READY
- Objective: Freeze MVP capability IDs for this wave from PHASE-06 and from the first-slice spec. This is an inventory, not a rewrite of discovery.
- In scope: Baseline artifact under `.cursor/docs/product-discovery/` (or `.cursor/docs/features/` if created). IDs for cadastro, agenda, OS, cobranca, WhatsApp copilot, timeline, Command OS guardrails.
- Out of scope: Domain WHAT contracts (PHASE-11), UX contracts (PHASE-12), code, using empty `README.md` as the MVP source.
- Hard dependencies: PHASE-06 `DONE`, PHASE-09 `DONE`
- Soft dependencies: first-slice spec (Human-authorized as wave contract input)
- Synchronization: none
- Blocking: none
- Tasks:
  - `TASK-01-freeze-mvp-capability-ids`
- Acceptance criteria:
  - Every first-slice command and success criterion has a stable ID.
  - EXCLUDED items from spec §12 are listed as out of this baseline.
  - No product code.
