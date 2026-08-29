# PHASE-12-ux-experience-contracts

- Horizon: MVP
- Specification state: DEFINED
- Execution state: READY
- Objective: Freeze shop-floor UX contracts. Do not remap journeys from scratch (PHASE-04 already did).
- In scope: UX contracts for recepção, tosador, and tutor WhatsApp. Pattern: contexto → ação → confirmação → próximo passo.
- Out of scope: Visual design system, app do tutor, traditional ERP IA, code.
- Hard dependencies: PHASE-10 `DONE`
- Soft dependencies: PHASE-11, PHASE-04
- Synchronization: none
- Blocking: none
- Tasks:
  - `TASK-01-contract-recepcao-tosador-shop-floor-ux`
  - `TASK-02-contract-tutor-whatsapp-ux`
- Acceptance criteria:
  - Screen persist path does not require LLM.
  - WhatsApp persist path requires confirmation v1 except `BaixarPagamento`.
  - No product code.
