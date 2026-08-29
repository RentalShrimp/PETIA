# TASK-01-write-canonical-product-specification

- Phase: PHASE-06-product-specification
- Horizon: DISCOVERY
- Specification state: DEFINED
- Execution state: READY
- Objective: Write the canonical product specification for this wave in one lot, sufficient to feed PHASE-10…12.
- In scope: `.cursor/docs/product-discovery/06-product-specification.md`
- Out of scope: Code, stack, PHASE-14, replacing the first-slice spec.
- Hard dependencies: PHASE-00…05 all `DONE`
- Soft dependencies: none
- Synchronization: none
- Allowed files: `.cursor/docs/product-discovery/06-product-specification.md`, `.cursor/docs/product-discovery/assumptions.md`
- Forbidden: product code
- Acceptance criteria:
  - Contains: users, entities, journeys summary, features, commands, rules (confirmation, isolation, idempotency), MVP cut, open Human decisions.
  - Meets `DECISION_AUTONOMY.md` §6 sufficient definition.
  - Does not authorize application code.
- Evidence: specification file exists and is internally consistent with prior discovery artifacts.
