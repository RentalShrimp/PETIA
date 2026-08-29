# TASK-01-model-tenant-unidade-tutor-pet-actors

- Phase: PHASE-01-business-model
- Horizon: DISCOVERY
- Specification state: DEFINED
- Execution state: READY
- Objective: Model actors and entities for this wave in one document: Tenant, Unidade, Usuario (recepção, tosador, dono), Tutor, Pet, VinculoTutorPet.
- In scope: `.cursor/docs/product-discovery/01-actors-and-entities.md`
- Out of scope: Holding operations, multi-tutor UX beyond “v1 one principal, model allows more”, code.
- Hard dependencies: PHASE-00 `DONE`
- Soft dependencies: none
- Synchronization: none
- Allowed files: `.cursor/docs/product-discovery/01-actors-and-entities.md`, `.cursor/docs/product-discovery/assumptions.md`
- Forbidden: product code
- Acceptance criteria:
  - Pet is first-class; tutor is not the aggregate root of the pet.
  - Independent shop = one Tenant + one Unidade; network uses the same model later.
  - Recepção and tosador are named as loja operators.
- Evidence: entity list with attributes needed for operational 360 (not LTV/churn).
