# TASK-04-contract-timeline-isolation-cross-domain

- Phase: PHASE-11-domain-capability-contracts
- Horizon: MVP
- Specification state: DEFINED
- Execution state: READY
- Objective: Write the cross-domain contract: event projection to pet timeline, comando_log SQL audit, isolation proofs required of later delivery. One lot after the three domain contracts.
- In scope: `.cursor/docs/product-discovery/11-contract-timeline-isolation.md`
- Out of scope: Code, analytics/intelligence.
- Hard dependencies: TASK-01, TASK-02, and TASK-03 of this Phase `DONE`
- Soft dependencies: none
- Synchronization: waits on TASK-01, TASK-02, TASK-03
- Allowed files: `.cursor/docs/product-discovery/11-contract-timeline-isolation.md`
- Forbidden: product code
- Acceptance criteria:
  - Timeline is a projection of events, not a hand-edited table.
  - Minimum events from spec §7 plus extras named in this Phase’s contracts.
  - Isolation tests required: other tenant cannot read/write; unidade mismatch rejected.
  - Same command from UI and chat produces the same facts.
- Evidence: event list, projection rules, required isolation cases.
