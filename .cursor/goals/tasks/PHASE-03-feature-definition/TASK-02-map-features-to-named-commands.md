# TASK-02-map-features-to-named-commands

- Phase: PHASE-03-feature-definition
- Horizon: DISCOVERY
- Specification state: DEFINED
- Execution state: READY
- Objective: Map every first-slice write feature to a named command from the spec catalog. One lot.
- In scope: `.cursor/docs/product-discovery/03-command-map.md`
- Out of scope: New commands not in the spec unless listed as a documented gap requiring Human; code.
- Hard dependencies: TASK-01 of this Phase `DONE`
- Soft dependencies: first-slice spec §5
- Synchronization: none
- Allowed files: `.cursor/docs/product-discovery/03-command-map.md`, `.cursor/docs/product-discovery/assumptions.md`
- Forbidden: product code; LLM-authored SQL as a feature
- Acceptance criteria:
  - Table: feature → command → chat confirmation v1 (yes/no) → actor.
  - All spec catalog commands appear exactly once as the write API.
  - Reads (agenda do dia, ficha, timeline) are marked query, not command.
- Evidence: complete map vs spec §5.
