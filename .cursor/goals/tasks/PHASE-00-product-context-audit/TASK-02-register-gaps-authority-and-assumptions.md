# TASK-02-register-gaps-authority-and-assumptions

- Phase: PHASE-00-product-context-audit
- Horizon: DISCOVERY
- Specification state: DEFINED
- Execution state: READY
- Objective: Register gaps, authority order, and auto-decided assumptions for DISCOVERY. One lot, not a questionnaire to the Human.
- In scope: `.cursor/docs/product-discovery/00-gaps-and-authority.md` plus assumptions in `.cursor/docs/product-discovery/assumptions.md` per `DECISION_AUTONOMY.md`.
- Out of scope: Asking the Human inferable questions, inventing reserved decisions, code.
- Hard dependencies: TASK-01 of this Phase `DONE`
- Soft dependencies: none
- Synchronization: none
- Allowed files: `.cursor/docs/product-discovery/00-gaps-and-authority.md`, `.cursor/docs/product-discovery/assumptions.md`
- Forbidden: product code; asking “how do you want this to work?”
- Acceptance criteria:
  - Authority chain is written: Goal > first-slice spec for this wave > discovery brainstorm > implementation plan proposal.
  - Unset Human decisions (stack, WhatsApp vendor, PSP, orchestrator runtime, pricing) are listed as gaps, not filled.
  - Every auto-decision follows the ask-gate and is recorded as ASSUMPTION.
- Evidence: artifacts exist; no new reserved decisions invented.
