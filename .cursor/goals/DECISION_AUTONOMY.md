# Decision Autonomy Protocol

Version: 1.0.0
Status: Operational
Review date: 2026-08-29

Applies to every **DISCOVERY** Phase and Task (`PHASE-00` … `PHASE-06`) when those files exist.

Product: **PETIA** (Pet Shop Operating System). Do not apply clinic/aesthetic-domain defaults from sibling products.

Product Tasks in later horizons inherit the same ask-gate when they would otherwise interrupt the Human for a low-impact product choice.

This file does not select work (`ORCHESTRATOR.md`) and does not mark units `DONE`.

---

## 1. Role of the Human

The Human is **Product Owner / strategic decision-maker**.

The agent discovers, structures, proposes, and documents.

The Human validates only what requires a human decision.

---

## 2. Decide without asking

The agent **must** decide alone when any of the following is true:

- the choice is implementation detail
- the choice is conventional UX for this domain
- a consolidated ERP/SaaS pattern exists
- the choice is easily reversible
- one option is clearly superior
- the answer can be inferred from documentation on disk
- the answer can be derived from pet shop operating processes
- the choice does not change product positioning

Record the decision as an **ASSUMPTION** in `.cursor/docs/product-discovery/assumptions.md` (create the file on first use).

```text
ASSUMPTION:
<decision>

RATIONALE:
<why this is standard, inferred, or reversible>

STATUS:
Auto-decided; reversible | Auto-decided; structural (cite DEC if already Accepted)
```

---

## 3. Ask the Human

The agent **must** ask when any of the following is true:

- the choice significantly changes positioning
- it conflicts with an already Accepted decision
- it has relevant financial impact (pricing, commercial model)
- it has legal/regulatory implication
- it is structurally hard to reverse
- two strategic alternatives are equally valid
- essential information cannot be inferred from disk or domain

### Before asking

1. Explain the problem
2. Present a recommendation
3. Explain why
4. Present alternatives only if they are actually relevant
5. Ask one objective question

Never ask: “How do you want this to work?” when a solution can be proposed.

Prefer: “I recommend X because Y. The alternative is Z. Because this affects positioning, I need validation: X or Z?”

---

## 4. Ask-gate (mandatory before any question)

```text
CAN I INFER THIS FROM DISK?
        ↓
YES → decide + document assumption
        ↓
NO
        ↓
IS IT A STANDARD DOMAIN PATTERN?
        ↓
YES → decide + document assumption
        ↓
NO
        ↓
IS IT REVERSIBLE?
        ↓
YES → decide + document assumption
        ↓
NO
        ↓
ASK USER (strategic form in §3)
```

Low-impact questions are forbidden. Replace them with explicit assumptions.

Do not ask for anything already present in `VISION.md`, `PRINCIPLES.md`, `DECISIONS.md`, `ARCHITECTURE.md`, `CONTRIBUTING.md`, `PETSHOP_OS_FEATURE_DISCOVERY.md`, `.cursor/docs/superpowers/specs/2026-08-29-petia-command-os-design.md`, `.cursor/docs/features/`, or prior DISCOVERY artifacts.

---

## 5. Discovery is not implementation

During DISCOVERY Phases:

**Forbidden:** product code, UI components, APIs, migrations, screens, stack choice.

**Allowed:** discover, model, specify, prioritize, document, check coherence, write artifacts under `.cursor/docs/product-discovery/`.

Implementation starts only after sufficient product definition exists **and** later MVP Phases plus architecture/slice gates allow it.

---

## 6. Sufficient product definition (exit)

DISCOVERY (`PHASE-00` … `PHASE-06`) may be `DONE` as a set only when:

- the domain is modeled
- primary users are defined
- primary journeys are mapped
- capabilities are identified
- main features are defined
- dependencies are identified
- critical rules are defined
- MVP is delimited
- gaps are identified
- pending strategic decisions are listed explicitly
- the specification can feed contract/implementation Phases without a new massive discovery round

Do not require absolute perfection. Target **sufficient product definition**.
