# PHASE-13-architecture-decision-gate

- Horizon: MVP
- Specification state: DEFINED
- Execution state: READY
- Objective: Prepare and obtain Human ratification of technical architecture while preserving Command OS. The superpowers implementation plan is a proposal, not Accepted.
- In scope: Architecture pack for Human: Command OS write path, ports (Agent, WhatsApp, Psp), isolation, what must appear in `ARCHITECTURE.md` / `DECISIONS.md`.
- Out of scope: Silently accepting Python/FastAPI/Postgres; writing application code; shipping PHASE-14.
- Hard dependencies: PHASE-10 `DONE`
- Soft dependencies: PHASE-11, PHASE-12
- Synchronization: none
- Blocking: Human must ratify stack in `ARCHITECTURE.md` / `DECISIONS.md` before PHASE-14 may start. This Phase may produce the pack while the decision is unset; it cannot mark architecture Accepted by itself.
- Tasks:
  - `TASK-01-prepare-command-os-architecture-for-ratification`
- Acceptance criteria:
  - Command OS contract is restated as architecture constraints (commands, handler SQL, orchestrator, no LLM SQL).
  - Open Human decisions (stack, WhatsApp vendor, PSP, orchestrator runtime) are listed, not invented.
  - No product code until Human ratifies; this Phase’s `DONE` means the pack exists and reserved decisions are explicitly listed. If the Human has not yet Accepted a stack, PHASE-14 remains blocked — this Phase may still be `DONE` as a gate pack.
