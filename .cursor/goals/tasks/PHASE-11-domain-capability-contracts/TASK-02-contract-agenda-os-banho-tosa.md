# TASK-02-contract-agenda-os-banho-tosa

- Phase: PHASE-11-domain-capability-contracts
- Horizon: MVP
- Specification state: DEFINED
- Execution state: READY
- Objective: Write WHAT contracts for serviço, profissional, agendamento, and ordem de serviço in one lot, including the named commands from agendar through saída.
- In scope: `.cursor/docs/product-discovery/11-contract-agenda-os.md`
- Out of scope: Dynamic duration, waitlist, equipment capacity, code, stack.
- Hard dependencies: PHASE-10 `DONE`
- Soft dependencies: TASK-01 of this Phase (pet_id exists)
- Synchronization: none (may start in parallel with TASK-01; must not assume unimplemented IDs)
- Allowed files: `.cursor/docs/product-discovery/11-contract-agenda-os.md`
- Forbidden: product code
- Acceptance criteria:
  - Commands: `Agendar`, `Reagendar`, `CancelarAgendamento`, `ConfirmarAgendamento`, `ConfirmarPresenca`, `AbrirOS`, `RegistrarEntradaOS`, `AtualizarExecucaoOS`, `RegistrarSaidaOS`.
  - Errors: `SLOT_OCUPADO`, `SERVICO_INEXISTENTE`.
  - Events: `AgendamentoCriado`, `AgendamentoConfirmado`, `PresencaConfirmada`, `OSAberta`, `OSEntradaRegistrada`, `OSConcluida`, plus `Reagendar` / `CancelarAgendamento` as honest extra event names if used.
  - RSVP ≠ check-in ≠ chat protocol confirmation.
- Evidence: payloads, statuses, overlap rule, OS JSON fields for entrada/execução/saída including fotos as URLs.
