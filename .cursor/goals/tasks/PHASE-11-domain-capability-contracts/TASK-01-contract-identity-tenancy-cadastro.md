# TASK-01-contract-identity-tenancy-cadastro

- Phase: PHASE-11-domain-capability-contracts
- Horizon: MVP
- Specification state: DEFINED
- Execution state: READY
- Objective: Write WHAT contracts for Tenant, Unidade, Usuario, Tutor, Pet, VinculoTutorPet and commands `RegistrarTutor`, `RegistrarPet`, `VincularPet` in one lot.
- In scope: `.cursor/docs/product-discovery/11-contract-identity-cadastro.md`
- Out of scope: Code, stack, agenda/OS.
- Hard dependencies: PHASE-10 `DONE`
- Soft dependencies: none
- Synchronization: none (parallelizable with TASK-02 and TASK-03)
- Allowed files: `.cursor/docs/product-discovery/11-contract-identity-cadastro.md`
- Forbidden: product code
- Acceptance criteria:
  - Isolation: every command/query carries `tenant_id`; loja operations carry `unidade_id`; `TENANT_MISMATCH` defined.
  - Pet-first attributes for operational 360 listed; LTV/churn excluded.
  - Payloads and events (`TutorRegistrado`, `PetRegistrado`) specified.
- Evidence: contract file with payloads, errors, isolation rules.
