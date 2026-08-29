# Goals Tree

Version: 1.0.0
Status: Operational
Review date: 2026-08-29

This file is the inventory and hierarchy map of the active Goal.

It does not execute work.
It does not review work.
It does not change `PROJECT_GOAL.md`.

---

## 1. Authority

```text
PROJECT_GOAL.md       → what may be built, In Scope, Out of Scope, DONE, STOP
GOALS_TREE.md         → what exists, how it is grouped, which horizon it belongs to
ORCHESTRATOR.md       → what to select next, what may run, what must wait, when to stop
DECISION_AUTONOMY.md  → DISCOVERY ask-gate (infer / assume / ask strategically)
EXECUTION_PROTOCOL.md → how to implement, test, recover, improve, complete a unit
REVIEW_PROTOCOL.md    → how to audit, verdict, compare before/after, reopen
```

`PROJECT_GOAL.md` is the highest authority. If this tree and the Goal conflict, the Goal wins for scope. This tree wins for physical inventory.

---

## 2. Physical layout

Root: `.cursor/goals/` relative to the open workspace.

```text
.cursor/goals/
├── PROJECT_GOAL.md
├── GOALS_TREE.md
├── ORCHESTRATOR.md
├── EXECUTION_PROTOCOL.md
├── REVIEW_PROTOCOL.md
├── DECISION_AUTONOMY.md
├── phases/
│   └── PHASE-XX-name.md
└── tasks/
    └── PHASE-XX-name/
        └── TASK-NN-name.md
```

Discovery must use the filesystem, not a remembered list.

Phase path: `.cursor/goals/phases/PHASE-XX-name.md`
Task path: `.cursor/goals/tasks/PHASE-XX-name/TASK-NN-name.md`

The Task directory name must equal the Phase filename stem.

Do not invent directories.
Do not invent Phase or Task files.
Do not treat a missing file as an existing unit.
Do not create `.cursor/goals/history/`. Cycle output in the session is the execution log.
Do not copy Bellia clinic Phase/Task names into this tree.

---

## 3. Hierarchy

```text
GOAL
  ↓
PHASE
  ↓
TASK
```

- One active Project Goal: `PROJECT_GOAL.md`
- A Phase belongs to the Goal
- A Task belongs to exactly one Phase
- Phase numbering is global and incremental (`00`, `01`, …)
- Task numbering is local to the Phase (`01`, `02`, …)
- Numeric order is a hint, not an absolute execution order

---

## 4. Discovery rules

On every orchestration cycle, rediscover from disk.

1. Read `PROJECT_GOAL.md`
2. List `phases/PHASE-*.md` sorted by `XX`
3. For the candidate Phase, list `tasks/PHASE-XX-name/TASK-*.md` sorted by `NN`
4. Ignore remembered names that are not on disk — including Bellia clinic units that existed in the source copy of these files

A Phase or Task file may be empty. Emptiness does not delete the unit. It means the contract is not yet written and must be inferred per `ORCHESTRATOR.md` and `EXECUTION_PROTOCOL.md`.

If `phases/` or `tasks/` is absent, there is no eligible Phase or Task. Do not invent work to compensate. Report the gap. Instantiation of this tree was Human-authorized on 2026-08-29; do not invent additional Phase/Task files beyond the inventory in §7.

---

## 5. Horizons

Every Phase has one horizon. A Task inherits the Phase horizon unless marked otherwise.

| Horizon | Meaning | Select under current Goal? |
|---|---|---|
| DISCOVERY | Product definition: context, domain, capabilities, features, journeys, MVP cut, spec | YES, when files exist |
| BOOTSTRAP | Enable autonomous execution itself | YES, when files exist |
| MVP | Contract and delivery work required for `PROJECT_GOAL.md` In Scope (Command OS first slice) | YES, when files exist |
| POST-MVP | Evolution listed in the tree but Out of Scope for this Goal | NO, unless Human authorizes |
| FUTURE | Advanced capabilities | NO, unless Human authorizes |

The Orchestrator may select only horizons allowed by the active Goal. This Goal allows `DISCOVERY ∪ BOOTSTRAP ∪ MVP`.

DISCOVERY Phases produce artifacts under `.cursor/docs/product-discovery/`. They must follow `DECISION_AUTONOMY.md`. They must not write product code.

BOOTSTRAP may run in parallel with DISCOVERY (no hard edge between the two spines). MVP contract Phases hard-depend on **both** sufficient product definition (PHASE-06 or Human-authorized first-slice spec) and the proven execution loop (PHASE-09), once those units exist.

Do not start POST-MVP or FUTURE work because it is technically interesting.

---

## 6. Current Goal mapping

Active Goal (from `PROJECT_GOAL.md`): **PETIA** — Pet Shop Operating System (HYPERION). This wave is the Command OS first slice for one unidade.

| Goal element | Phase (on disk) |
|---|---|
| Canonical product context (VISION, discovery, first-slice spec, gaps) | PHASE-00 |
| Petshop business / domain processes (tutor, pet, unidade, serviços) | PHASE-01 |
| Product capabilities (not yet MVP-cut) | PHASE-02 |
| Coherent features from capabilities | PHASE-03 |
| End-to-end journeys (loja, recepção, tosador, tutor no WhatsApp) | PHASE-04 |
| MVP / POST-MVP / FUTURE / EXCLUDED cut | PHASE-05 |
| Canonical product specification (feeds contracts) | PHASE-06 |
| Repository reality / no assumed modules | PHASE-07 |
| Execution structure | PHASE-08, PHASE-09 |
| MVP capability inventory and wave baseline (IDs from spec) | PHASE-10 |
| Per-domain WHAT contracts for the first slice | PHASE-11 |
| End-to-end UX contracts (shop floor, not traditional ERP) | PHASE-12 |
| Ratify technical architecture before code delivery — must preserve Command OS | PHASE-13 |
| First Human-authorized MVP slice with evidence (incl. Command OS guardrails) | PHASE-14 |

**Non-duplication:** PHASE-00…06 discover and specify. PHASE-10…12 freeze IDs and executable contracts from the spec and from `.cursor/docs/superpowers/specs/2026-08-29-petia-command-os-design.md`. PHASE-12 does **not** remap journeys from scratch (PHASE-04 already did). PHASE-10 does **not** treat an empty `README.md` as the sole MVP source once PHASE-06 or the first-slice spec exists.

**Intelligence mapping:** Inteligência is a **transversal overlay** (LLM as client of the command catalog), not a chatbot Phase. Cadastro, agenda, OS, cobrança, and WhatsApp remain first-class domain contracts in PHASE-11; the autonomy matrix encodes command → confirmation → handler, never LLM → SQL.

Product **implementation** (PHASE-14) remains gated by ratified architecture (PHASE-13) and a Human-named first slice. Changing the Goal text alone does **not** authorize code delivery. DISCOVERY `DONE` is not code authorization. The first-slice spec is the wave contract, not a license to skip PHASE-13.

---

## 7. Inventory

Exact inventory on disk as of 2026-08-29 (Human-authorized instantiation). Do not invent units not listed here.

### On disk (control plane)

```text
.cursor/goals/
├── PROJECT_GOAL.md
├── GOALS_TREE.md
├── ORCHESTRATOR.md
├── EXECUTION_PROTOCOL.md
├── REVIEW_PROTOCOL.md
└── DECISION_AUTONOMY.md
```

### Product sources on disk (not Phases)

- `VISION.md`, `PRINCIPLES.md`, `DECISIONS.md`, `ARCHITECTURE.md`, `CONTRIBUTING.md` (canonical; several are stubs)
- `PETSHOP_OS_FEATURE_DISCOVERY.md` (brainstorm; 22 domains; not an MVP backlog)
- `.cursor/docs/superpowers/specs/2026-08-29-petia-command-os-design.md` (first-slice contract)
- `.cursor/docs/superpowers/plans/2026-08-29-petia-command-os.md` (implementation plan; stack proposal, not ratification; **not** the PHASE-14 execution script)

### DISCOVERY / BOOTSTRAP / MVP Phase files (on disk)

```text
phases/PHASE-00-product-context-audit.md
phases/PHASE-01-business-model.md
phases/PHASE-02-capability-discovery.md
phases/PHASE-03-feature-definition.md
phases/PHASE-04-ux-user-journeys.md
phases/PHASE-05-mvp-prioritization.md
phases/PHASE-06-product-specification.md
phases/PHASE-07-repository-audit.md
phases/PHASE-08-project-structure.md
phases/PHASE-09-execution-system.md
phases/PHASE-10-mvp-product-baseline.md
phases/PHASE-11-domain-capability-contracts.md
phases/PHASE-12-ux-experience-contracts.md
phases/PHASE-13-architecture-decision-gate.md
phases/PHASE-14-first-mvp-slice-delivery.md
```

### Task files (on disk)

```text
tasks/PHASE-00-product-context-audit/
  TASK-01-audit-vision-discovery-and-first-slice-spec.md
  TASK-02-register-gaps-authority-and-assumptions.md
tasks/PHASE-01-business-model/
  TASK-01-model-tenant-unidade-tutor-pet-actors.md
  TASK-02-model-agenda-banho-tosa-whatsapp-cobranca-processes.md
tasks/PHASE-02-capability-discovery/
  TASK-01-inventory-capabilities-from-feature-discovery.md
  TASK-02-bound-first-slice-capabilities-command-os.md
tasks/PHASE-03-feature-definition/
  TASK-01-define-features-from-capabilities.md
  TASK-02-map-features-to-named-commands.md
tasks/PHASE-04-ux-user-journeys/
  TASK-01-map-recepcao-tosador-loja-journeys.md
  TASK-02-map-tutor-whatsapp-copilot-journeys.md
tasks/PHASE-05-mvp-prioritization/
  TASK-01-cut-mvp-post-mvp-future-excluded.md
tasks/PHASE-06-product-specification/
  TASK-01-write-canonical-product-specification.md
  TASK-02-trace-specification-to-first-slice-spec.md
tasks/PHASE-07-repository-audit/
  TASK-01-audit-repository-reality.md
tasks/PHASE-08-project-structure/
  TASK-01-define-execution-and-docs-layout.md
tasks/PHASE-09-execution-system/
  TASK-01-prove-orchestrator-can-select-from-disk.md
tasks/PHASE-10-mvp-product-baseline/
  TASK-01-freeze-mvp-capability-ids.md
tasks/PHASE-11-domain-capability-contracts/
  TASK-01-contract-identity-tenancy-cadastro.md
  TASK-02-contract-agenda-os-banho-tosa.md
  TASK-03-contract-cobranca-whatsapp-command-os.md
  TASK-04-contract-timeline-isolation-cross-domain.md
tasks/PHASE-12-ux-experience-contracts/
  TASK-01-contract-recepcao-tosador-shop-floor-ux.md
  TASK-02-contract-tutor-whatsapp-ux.md
tasks/PHASE-13-architecture-decision-gate/
  TASK-01-prepare-command-os-architecture-for-ratification.md
tasks/PHASE-14-first-mvp-slice-delivery/
  TASK-01-deliver-command-os-core-cadastro-agenda-os.md
  TASK-02-deliver-cobranca-copilot-shop-and-pilot-evidence.md
```

Specification state of these units: `DEFINED`. Execution state: `READY` (none `DONE`). Do not add Bellia clinic names. Do not add POST-MVP Phases without Human authorization.

### POST-MVP / FUTURE

No Phase or Task files on disk. Goal Out of Scope / VISION expansion (holding, estoque, planos, delivery, CRM de campanha, copiloto executivo, app do tutor, NFe, intelligence preditiva) remains **uninventoried** — do not select or create these units without Human authorization.

---

## 8. Dependency types

Dependencies are conceptual. Written contracts in Phase/Task files win over this map.

### Hard

The dependent Task must not start until the producer is `DONE` (or reopened and re-completed).

### Soft

The Task may start. Another Task provides useful context. Missing the soft dependency is not a blocker.

### Synchronization

Two or more Tasks may run independently. A later Task must wait until all of them are `DONE`.

### Blocking

A failure, `BLOCKED` state, or unresolved reserved decision in Task A prevents Task B.

### Reopen

A later discovery may require a completed Task to leave `DONE` and be reworked. `DONE` is not immutable.

---

## 9. Inferred Phase dependencies

Phase files are on disk. The map below is the spine. Written hard/soft/sync fields in Phase/Task files win over this map.

```text
DISCOVERY:  PHASE-00 → PHASE-01 → PHASE-02 → PHASE-03 → PHASE-04 → PHASE-05 → PHASE-06
BOOTSTRAP:  PHASE-07 → PHASE-08 → PHASE-09
JOIN:       (PHASE-06 AND PHASE-09) → PHASE-10 ─┬─→ PHASE-11 ─┐
                                                ├─→ PHASE-12 ─┼─→ PHASE-14
                                                └─→ PHASE-13 ─┘
```

Hard spine (simplified):

```text
PHASE-00 → PHASE-01 → PHASE-02 → PHASE-03 → PHASE-04 → PHASE-05 → PHASE-06 ─┐
PHASE-07 → PHASE-08 → PHASE-09 ─────────────────────────────────────────────┴→ PHASE-10
                                                                                    ├─→ PHASE-11 ─┐
                                                                                    ├─→ PHASE-12 ─┼─→ PHASE-14
                                                                                    └─→ PHASE-13 ─┘
```

- PHASE-01 hard-depends on PHASE-00
- PHASE-02 hard-depends on PHASE-01
- PHASE-03 hard-depends on PHASE-02
- PHASE-04 hard-depends on PHASE-03
- PHASE-05 hard-depends on PHASE-03 and PHASE-04
- PHASE-06 hard-depends on PHASE-00…05 (`DONE`)
- PHASE-08 hard-depends on PHASE-07
- PHASE-09 hard-depends on PHASE-07 and PHASE-08
- PHASE-10 hard-depends on PHASE-06 and PHASE-09 (spec + proven loop). Soft: first-slice spec already on disk may inform PHASE-10 once Human authorizes consuming it.
- PHASE-11 hard-depends on PHASE-10 (`DONE`, not merely TASK-01)
- PHASE-12 hard-depends on PHASE-10 (soft on PHASE-11; soft on PHASE-04 journeys)
- PHASE-13 hard-depends on PHASE-10 (soft on PHASE-11/12); ratification additionally blocked on Human stack/architecture decision
- PHASE-14 hard-depends on PHASE-11, PHASE-12, and PHASE-13; first delivery additionally blocked on Human-named first slice

Do not treat numeric order as a substitute for this map.

---

## 10. Inferred Task dependencies inside a Phase

Task files are on disk with written contracts. Those written hard/soft/sync fields win. Default inference applies only if a body is empty:

1. Inventory / model before consumers
2. Implementation before tests / evaluation / audit
3. Numeric order is the tie-breaker only among Tasks with equal inferred priority and no hard dependency

Do not block a Task solely because a later-numbered Task exists.

When Task files are created, write hard/soft/sync fields in each file. Those written fields win over inference.

---

## 11. Parallel sets

A set is parallelizable when every member is eligible, none has a hard dependency on another member, and no two members must write the same protected contract.

One agent session still executes one Task at a time. Parallelism means selection may choose any eligible member, and multiple agents may be assigned different members if the Human runs them.

Parallel sets (when each member is eligible):

- DISCOVERY spine vs BOOTSTRAP spine (PHASE-00…06 vs PHASE-07…09)
- After PHASE-10 `DONE`: PHASE-11 TASK-01, TASK-02, TASK-03 (TASK-04 waits on those three)
- After PHASE-10 `DONE`: PHASE-12 TASK-01 and TASK-02
- PHASE-11 and PHASE-12 as Phases after PHASE-10

One agent session still executes one Task at a time.

---

## 12. Phase / Task specification state

| Concept | Applies to | Example values |
|---|---|---|
| Specification state | the written contract | `DEFINED`, `DECOMPOSED`, `REVIEWED`, `APPROVED` |
| Execution state | Orchestrator / Builder progress | `READY`, `IN_PROGRESS`, `DONE`, `BLOCKED` |

A populated Phase or Task file means the work has been specified. It does not mean the work has been executed.

Phase and Task bodies exist and are `DEFINED`. Execution has not started. Do not create additional Phase or Task files to compensate. If a future empty body appears, infer only inside Goal In Scope; reserved Human Decisions still block eligibility.

---

## 13. Instantiation (this repository)

This repository **copied** the Cursor Goal kit from Bellia and, on 2026-08-29, rebound the control files to PETIA.

- Protocols remain the execution motor; binds are factual to this workspace.
- `PROJECT_GOAL.md` and this tree describe PETIA and the inventory on disk.
- Control files live under `.cursor/goals/` (this repo). Do not look for `.cursor/goal/`.
- Phase/Task tree **is** instantiated (PETIA names, 2026-08-29 Human authorization). Do not treat remembered Bellia clinic units as existing.
- DISCOVERY sources already on disk (`PETSHOP_OS_FEATURE_DISCOVERY.md`, first-slice spec) are inputs, not Phase completion.
- `DECISION_AUTONOMY.md` governs DISCOVERY questions vs assumptions.
- Do not copy domain trees from sibling products.
- Do not overwrite Phase/Task inventory without Human authorization.
- Do not treat Goal rewrite or DISCOVERY `DONE` as permission to implement domain features; PHASE-14 still requires architecture ratification and a Human-named first slice.

---

## 14. What this file must never do

- Select the next Task
- Mark a Task `DONE`
- Authorize product-domain work solely because the Goal text changed
- Modify Phase or Task files
- Modify `PROJECT_GOAL.md`
- Create evidence stores, scripts, or extra control files
- Invent Bellia clinic Phases or select work that is not on disk
