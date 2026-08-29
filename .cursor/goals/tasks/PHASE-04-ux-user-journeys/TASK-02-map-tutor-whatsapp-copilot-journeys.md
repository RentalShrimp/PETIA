# TASK-02-map-tutor-whatsapp-copilot-journeys

- Phase: PHASE-04-ux-user-journeys
- Horizon: DISCOVERY
- Specification state: DEFINED
- Execution state: READY
- Objective: Map tutor WhatsApp copilot journeys in one lot: agendar, reagendar, cancelar, RSVP, pet pronto, cobrança, recomendar; confirmation protocol; agent-down fallback.
- In scope: `.cursor/docs/product-discovery/04-whatsapp-journeys.md`
- Out of scope: App/portal, NFe, code.
- Hard dependencies: TASK-01 of this Phase `DONE`
- Soft dependencies: none
- Synchronization: none
- Allowed files: `.cursor/docs/product-discovery/04-whatsapp-journeys.md`, `.cursor/docs/product-discovery/assumptions.md`
- Forbidden: product code
- Acceptance criteria:
  - Every mutating chat path shows propose → “Confirmo …?” → sim → same command as loja.
  - Unknown tutor does not create a ghost cadastro without confirmation.
  - Fora do catálogo is an explicit refusal.
  - Agent timeout: message that recepção can operate the screen.
- Evidence: journey steps including confirmation and failure paths.
