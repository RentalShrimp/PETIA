# TASK-02-contract-tutor-whatsapp-ux

- Phase: PHASE-12-ux-experience-contracts
- Horizon: MVP
- Specification state: DEFINED
- Execution state: READY
- Objective: Freeze WhatsApp copy and confirmation UX for the tutor in one lot.
- In scope: `.cursor/docs/product-discovery/12-ux-whatsapp.md`
- Out of scope: Meta Cloud API details, code, loja screens.
- Hard dependencies: PHASE-10 `DONE`
- Soft dependencies: PHASE-04 TASK-02, PHASE-11 TASK-03
- Synchronization: none
- Allowed files: `.cursor/docs/product-discovery/12-ux-whatsapp.md`
- Forbidden: product code
- Acceptance criteria:
  - Confirmation prompt pattern includes the spec example shape (“Confirmo o banho do Thor sábado 11:30?”) when data exists.
  - Sim/não/expirado/duplicado behaviors specified.
  - Refusal copy for fora do catálogo and agent-down.
- Evidence: message templates tied to commands.
