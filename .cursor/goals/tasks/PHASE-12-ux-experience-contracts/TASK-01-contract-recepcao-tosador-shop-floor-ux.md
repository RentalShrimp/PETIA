# TASK-01-contract-recepcao-tosador-shop-floor-ux

- Phase: PHASE-12-ux-experience-contracts
- Horizon: MVP
- Specification state: DEFINED
- Execution state: READY
- Objective: Freeze shop-floor UX contract for recepção and tosador in one lot. Consumes PHASE-04 journeys; does not remap from scratch.
- In scope: `.cursor/docs/product-discovery/12-ux-loja.md`
- Out of scope: Visual design system, code, WhatsApp copy (TASK-02).
- Hard dependencies: PHASE-10 `DONE`
- Soft dependencies: PHASE-04 TASK-01, PHASE-11
- Synchronization: none (parallelizable with TASK-02)
- Allowed files: `.cursor/docs/product-discovery/12-ux-loja.md`
- Forbidden: product code; traditional ERP navigation as the primary pattern
- Acceptance criteria:
  - Surfaces: agenda do dia, ficha pet/timeline, OS entrada/execução/saída.
  - Persist via `POST` command equivalent, never LLM.
  - Check-in is one action on the screen.
- Evidence: screen → actions → commands table.
