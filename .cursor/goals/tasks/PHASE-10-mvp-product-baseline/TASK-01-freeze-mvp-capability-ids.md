# TASK-01-freeze-mvp-capability-ids

- Phase: PHASE-10-mvp-product-baseline
- Horizon: MVP
- Specification state: DEFINED
- Execution state: READY
- Objective: Freeze stable capability IDs for the first slice in one lot. Consume PHASE-06 and the first-slice spec. Do not rediscover.
- In scope: `.cursor/docs/product-discovery/10-mvp-capability-ids.md`
- Out of scope: WHAT contracts (PHASE-11), code, README as MVP source.
- Hard dependencies: PHASE-06 `DONE`, PHASE-09 `DONE`
- Soft dependencies: first-slice spec
- Synchronization: none
- Allowed files: `.cursor/docs/product-discovery/10-mvp-capability-ids.md`
- Forbidden: product code
- Acceptance criteria:
  - IDs exist for: identity/tenancy, cadastro, agenda, OS, cobranca, WhatsApp copilot, timeline, Command OS guardrails, shop-floor UX.
  - Each ID traces to a spec section and/or PHASE-06 heading.
  - Excluded §12 items have IDs marked OUT_OF_WAVE.
- Evidence: ID catalog complete and unique.
