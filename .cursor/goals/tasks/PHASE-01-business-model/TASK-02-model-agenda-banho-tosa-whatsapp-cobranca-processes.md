# TASK-02-model-agenda-banho-tosa-whatsapp-cobranca-processes

- Phase: PHASE-01-business-model
- Horizon: DISCOVERY
- Specification state: DEFINED
- Execution state: READY
- Objective: In one lot, document the operational processes of the pilot week: agenda (fixed duration), OS banho & tosa (entrada → execução → saída), WhatsApp as channel, PIX/link settlement.
- In scope: `.cursor/docs/product-discovery/01-operational-processes.md`
- Out of scope: Dynamic duration, waitlist, estoque, recurring plans, code.
- Hard dependencies: TASK-01 of this Phase `DONE`
- Soft dependencies: none
- Synchronization: none
- Allowed files: `.cursor/docs/product-discovery/01-operational-processes.md`, `.cursor/docs/product-discovery/assumptions.md`
- Forbidden: product code
- Acceptance criteria:
  - Process matches spec loop: cadastro → agendar → RSVP → presença → OS → cobrança → timeline.
  - Distinguishes protocol confirmation (chat “sim”) from domain RSVP and check-in.
  - `BaixarPagamento` is a system/PSP process, not a chat confirmation.
- Evidence: process narrative with actors at each step.
