# PHASE-01-business-model

- Horizon: DISCOVERY
- Specification state: DEFINED
- Execution state: READY
- Objective: Model the petshop domain this Goal needs: Tenant, Unidade, tutor, pet, recepção, tosador, agenda, banho & tosa, WhatsApp, cobrança — as business processes, not modules.
- In scope: Domain model and process narratives under `.cursor/docs/product-discovery/`. Pet-first. One unidade in v1; Tenant → Unidade in the model.
- Out of scope: Holding/franquia operations, estoque, planos, delivery, CRM de campanha, code, stack.
- Hard dependencies: PHASE-00 `DONE`
- Soft dependencies: none
- Synchronization: none
- Blocking: none
- Tasks:
  - `TASK-01-model-tenant-unidade-tutor-pet-actors`
  - `TASK-02-model-agenda-banho-tosa-whatsapp-cobranca-processes`
- Acceptance criteria:
  - Actors, entities, and the pilot-week operational processes are documented.
  - Pet is a first-class entity, not a field on the tutor.
  - No product code.
