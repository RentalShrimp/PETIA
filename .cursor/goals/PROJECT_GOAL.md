# PROJECT GOAL — PETIA

Version: 1.0.0
Status: Active
Review date: 2026-08-29

## Objective

Build **PETIA**, the Pet Shop Operating System (HYPERION): a vertical OS that evolves **Record → Action → Intelligence**, with the **pet as a first-class entity**.

This Goal wave delivers the **Command OS first slice** for one unit: operational 360 cadastro (tutor + pet), agenda, ordem de serviço de banho & tosa, copiloto no WhatsApp, cobrança PIX/link with automatic settlement, and pet timeline.

An agent with no prior conversation can answer: Goal, current Phase, eligible Task, how to validate, how to record evidence, and what happens next.

## Target State

```text
Open workspace
  → read PROJECT_GOAL.md
  → discover Phases and Tasks from disk
  → select one eligible Task
  → execute (inspect → plan → implement → test → review → improve)
  → route verdict (OBJECTIVE_PASS | APPROVED | REJECTED | BLOCKED)
  → reopen when new evidence falsifies DONE
  → continue or stop
```

The loop content is **PETIA product work** (docs, architecture when ratified, then software) — not the reusable kit itself, and not a copy of sibling products.

## In Scope

- Product definition already on disk: `VISION.md`, `PRINCIPLES.md`, `DECISIONS.md`, `ARCHITECTURE.md`, `CONTRIBUTING.md`, `PETSHOP_OS_FEATURE_DISCOVERY.md`
- First-slice design: `.cursor/docs/superpowers/specs/2026-08-29-petia-command-os-design.md`
- Product Discovery Phases (context, domain, capabilities, features, journeys, MVP cut, specification) under `.cursor/goals/` when those files exist, producing artifacts in `.cursor/docs/product-discovery/`
- Control plane under `.cursor/goals/` for this repository, including `DECISION_AUTONOMY.md`
- BOOTSTRAP Phases (repository audit, structure, execution loop) as execution-enablement only, when those files exist
- Domain contract and delivery tree that **consumes** the product specification and the first-slice spec instead of rediscovering or copying Bellia
- Future product implementation that stays inside ratified decisions and this Goal’s In Scope
- Command OS model: named commands are the product; the LLM is a client of the catalog — not LLM-only product, not mute ERP
- Review, reopen, anti-fake-completion, Human Intervention

## Out of Scope

- Treating this repository as a reusable “execution OS kit” destination for other products
- Copying domain Phases/Tasks from sibling repositories (including Bellia clinic OS)
- Implementing the 22 discovery domains in `PETSHOP_OS_FEATURE_DISCOVERY.md` as if they were this Goal wave
- Holding, franquia multiunidade, estoque, compras, planos/assinatura, delivery, CRM de campanha, LTV/churn, copiloto executivo, portal/app do tutor, NFe, folha, or intelligence preditiva as this wave (spec §12)
- Choosing or implementing a **second** stack that contradicts an Accepted architecture decision, or shipping product code before a Human-named first slice and ratified architecture
- Competing as a lowest-price basic agenda/cadastro product
- Creating a state database, evidence directory, or control script for the execution loop
- Declaring definitive pricing, stack, WhatsApp vendor, or PSP vendor as if Accepted while they remain unset
- Treating the Human as feature specifier for reversible or inferable product choices (`DECISION_AUTONOMY.md`)
- Product work that requires a reserved Human Decision still listed as unset
- Rewriting the MVP as an LLM/chatbot-only product that replaces cadastro, agenda, OS, and cobrança domain capabilities
- Letting the LLM compose SQL, skip chat confirmation (v1 default), or hold privileged authority (tenancy, permissions, financial execution)
- Exposing internal multi-agent architecture as a mandatory user-facing product concept without need

## Architectural Constraints

- Discovery uses the filesystem of the open workspace, never a remembered list.
- There is exactly one Goal root: `.cursor/goals/` relative to the open workspace.
- Persistent execution state is not stored in a database. Report state in the cycle output.
- `DONE` requires objective, acceptance criteria, evidence, review `APPROVED`, and the improvement pass (or a recorded skip).
- `IMPLEMENT → DONE` is forbidden.
- `OBJECTIVE_PASS` is never Task completion.
- `DONE` is reversible by reopen. `CANCELLED` is Human-only.
- Product Tasks must not modify protected control files without authorization.
- DISCOVERY Tasks follow `DECISION_AUTONOMY.md`: infer, assume, or ask strategically — never treat the Human as the feature specifier.
- DISCOVERY must not create product code, UI, APIs, or choose a stack.
- Do not assume modules, stacks, or files that are not on disk.
- Do not invent reserved Human Decisions.
- **Pet-first.** Pet is a first-class entity, not a field on the tutor.
- **One write path.** Screen and WhatsApp dispatch the same named command catalog (`tenant_id`; store-day operations also `unidade_id`).
- **LLM only in chat.** The shop screen does not converse in order to persist. The orchestrator is required for agent calls and is not the owner of business rules.
- **SQL only in the handler**, parameterized. The model must not compose SQL. Audit may show the parameterized SQL the handler will run.
- **Isolation:** every query and command is tenant-scoped; agenda and OS do not cross unidade in this wave.
- **Chat v1:** persist only after confirmation, unless the store later configures per-command level via `ConfigurarNivelAcao`.
- Technical architecture in `ARCHITECTURE.md` is a **stub**, not Accepted. The Command OS contract in the first-slice spec binds this wave. Do not invent a second stack once one is Accepted in `DECISIONS.md` / `ARCHITECTURE.md`. The implementation plan’s stack proposal is not ratification.
- Canonical directional docs at repo root bind narrative as they are filled: `VISION.md`, `PRINCIPLES.md`, `DECISIONS.md`, `CONTRIBUTING.md`. Do not treat empty sections as Accepted decisions.
- UX must not regress to traditional ERP navigation when context allows `contexto → ação → confirmação → próximo passo`.

## Execution Model

```text
INSPECT → UNDERSTAND → PLAN → IMPLEMENT → TEST → VALIDATE → REVIEW
  → OBJECTIVE_PASS → IMPROVE → APPROVED → DONE
```

DISCOVERY Phases define the product (docs only). BOOTSTRAP Phases enable the loop; they are not PETIA domain delivery.

Phase/Task files **are on disk** (Human-authorized instantiation 2026-08-29). Discover them from `.cursor/goals/phases/` and `.cursor/goals/tasks/`. Do not copy Bellia clinic Phases. Do not invent extra units beyond `GOALS_TREE.md` inventory.

Product implementation of the first slice starts only after:

1. this Goal describes PETIA (this file),
2. DISCOVERY has reached sufficient product definition **or** the Human has authorized consuming the first-slice spec as the wave contract,
3. domain Phase/Task inventory exists on disk,
4. architecture is ratified (`ARCHITECTURE.md` / `DECISIONS.md`) and the Human has named the first MVP slice.

The first-slice spec already names that slice. Naming it in the spec does **not** by itself authorize writing application code.

## HUMAN DECISION REQUIRED (unset)

- Ratify technical stack in `ARCHITECTURE.md` / `DECISIONS.md` (the implementation plan proposes Python 3.12, FastAPI, PostgreSQL 16 — that is a proposal, not Accepted).
- Concrete WhatsApp / messaging vendor.
- Concrete PSP / PIX vendor.
- Concrete agent-orchestrator implementation (port exists in the spec; vendor/runtime unset).
- Definitive pricing and plans.
- Whether `AGENTS.md` (or equivalent agent catalog) is required for this repository.
- Filling canonical docs (`VISION.md`, `PRINCIPLES.md`, `DECISIONS.md`, `ARCHITECTURE.md`) beyond their current stubs.
- Authorization to write application code (PHASE-14), distinct from tree instantiation.

Do not invent those decisions here.

## HUMAN DECISION RATIFIED

- PETIA is the Pet Shop Operating System, proprietary HYPERION intellectual property (`VISION.md`, `CONTRIBUTING.md`).
- Dual mission: real product (pilot store operates a week of banho & tosa on PETIA) and public build (YouTube episode shows that same loop).
- Cliente: rede/franquia **and** loja independente; same system. v1 operates **one unidade** excellently; model is Tenant → Unidade from day 1.
- First slice frozen in `.cursor/docs/superpowers/specs/2026-08-29-petia-command-os-design.md`: cadastro 360 operacional, agenda, OS banho & tosa, WhatsApp copilot, PIX/link with webhook settlement, pet timeline.
- Architecture of this wave: **Command OS** (not Workflow OS, not free Agent-SQL). Command catalog is the product; LLM is a client.
- Pet-first. Screen and WhatsApp share the same commands. Chat v1 confirms before persist. SQL only in the handler.
- Success of this wave: the pilot store uses the loop for real **and** the episode films that loop.
- 2026-08-29 — Humano authorized rebinding these control files from a Bellia copy so they describe PETIA, this workspace, and the inventory on disk. That authorization is not permission to implement product code.
- 2026-08-29 — Humano authorized instantiation of the PETIA Phase/Task tree on disk (large-batch Tasks; do not execute the superpowers implementation plan as the execution script). That authorization is not permission to implement product code (PHASE-14 still requires ratified architecture and a named code slice).

## Project DONE

- [ ] The control files describe **PETIA** (this repository), with authority order intact (`PROJECT_GOAL.md`, `GOALS_TREE.md`, `ORCHESTRATOR.md`, `DECISION_AUTONOMY.md`, `EXECUTION_PROTOCOL.md`, `REVIEW_PROTOCOL.md`).
- [ ] DISCOVERY Phase/Task tree exists on disk and can reach sufficient product definition without treating the Human as feature specifier — **or** the Human has explicitly authorized the first-slice spec as the wave contract and remaining discovery is tracked as later-horizon.
- [x] Domain Phase/Task tree for the PETIA first slice exists on disk (PETIA names; no Bellia clinic leftover).
- [ ] Technical architecture is either ratified in `DECISIONS.md` / `ARCHITECTURE.md` or every remaining In-Scope path is explicitly blocked on that decision (no silent stack choice).
- [ ] First-slice capabilities from the Command OS spec that the Human has authorized for this Goal wave are implemented with evidence (not merely documented): cadastro, agenda, OS, copiloto, cobrança, timeline.
- [ ] Multi-tenant identity/permissions and unidade isolation required by the spec are proven where persistence/API exist.
- [ ] Command OS guardrails are evidenced for any automated path that ships: named commands, confirmation in chat v1, no LLM SQL, no LLM privileged authority on tenancy, permissions, or financial execution.
- [ ] UX for shipped surfaces follows shop-floor action (not traditional-ERP regression) where context allows direct action.
- [ ] Cold discovery still locates Goal, Phases, and Tasks from disk alone.
- [ ] `IMPLEMENT → DONE` remains unreachable; `OBJECTIVE_PASS` is never treated as Task `DONE`.
- [ ] No reserved unset Human Decision is silently assumed in order to claim Goal `DONE`.

## STOP CONDITION

Declare this Goal `DONE` only when every checkbox above is satisfied with evidence and reviewed.

If a reserved Human Decision is required and unset, declare `BLOCKED`, record the decision needed, and stop.

Do not declare `DONE` because files were written.
Do not declare `DONE` because DISCOVERY or BOOTSTRAP Phases finished or because domain contracts were merely authored.
Do not declare `DONE` because a sibling product (Bellia) had a later Phase `DONE`.

## REVIEW

BUILDER: implements, validates, and produces evidence.
REVIEWER: tries to prove the work is not `DONE`, auditing criteria, scope, tests, evidence, and protected files.
