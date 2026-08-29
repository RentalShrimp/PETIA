# TASK-01-prepare-command-os-architecture-for-ratification

- Phase: PHASE-13-architecture-decision-gate
- Horizon: MVP
- Specification state: DEFINED
- Execution state: READY
- Objective: Produce the architecture ratification pack in one lot. Preserve Command OS. Do not Accept a stack. Do not implement.
- In scope: `.cursor/docs/product-discovery/13-architecture-pack.md`. May propose options (including the superpowers plan’s Python/FastAPI/Postgres) clearly labeled PROPOSAL.
- Out of scope: Writing `DECISIONS.md` Accepted stack without Human; application code; executing the implementation plan.
- Hard dependencies: PHASE-10 `DONE`
- Soft dependencies: PHASE-11, PHASE-12
- Synchronization: none
- Blocking: Completing this Task does not unlock PHASE-14 by itself. PHASE-14 remains blocked until Human Accepts stack in `DECISIONS.md` / `ARCHITECTURE.md` and names the first slice for code.
- Allowed files: `.cursor/docs/product-discovery/13-architecture-pack.md`
- Forbidden: product code; treating the implementation plan as ratified; inventing a second write path
- Acceptance criteria:
  - Restates: named commands, handler parameterized SQL, orchestrator for agent only, LLM client, ports Agent/WhatsApp/Psp, webhook baixa, tenant/unidade isolation.
  - Lists reserved Human decisions still unset.
  - States that `.cursor/docs/superpowers/plans/2026-08-29-petia-command-os.md` is not the execution unit for PHASE-14.
- Evidence: pack exists; no silent stack Accept.
