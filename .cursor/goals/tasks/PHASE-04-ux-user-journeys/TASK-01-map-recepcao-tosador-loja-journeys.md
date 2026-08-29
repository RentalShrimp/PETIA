# TASK-01-map-recepcao-tosador-loja-journeys

- Phase: PHASE-04-ux-user-journeys
- Horizon: DISCOVERY
- Specification state: DEFINED
- Execution state: READY
- Objective: Map loja journeys for recepção and tosador for the entire pilot-week loop in one artifact.
- In scope: `.cursor/docs/product-discovery/04-loja-journeys.md`
- Out of scope: WhatsApp tutor journeys (TASK-02), pixel UI, code, ERP menu taxonomy.
- Hard dependencies: PHASE-03 `DONE`
- Soft dependencies: none
- Synchronization: none
- Allowed files: `.cursor/docs/product-discovery/04-loja-journeys.md`, `.cursor/docs/product-discovery/assumptions.md`
- Forbidden: product code
- Acceptance criteria:
  - Recepção: cadastro, agenda, check-in, abrir OS, emitir cobrança.
  - Tosador: entrada (fotos/checklist), execução, saída (fotos, próxima visita, recomendar).
  - Screen writes are commands, never chat-with-LLM-to-save.
  - UX pattern: contexto → ação → confirmação operacional (toque) → próximo passo.
- Evidence: step lists with actor and command name per step.
