# Orchestrator

Version: 1.0.0
Status: Operational
Review date: 2026-08-29

The Orchestrator is the control plane of autonomous development.

Product: **PETIA** (Pet Shop Operating System). Discovery, selection, and stop conditions are relative to this repository.

It decides what work exists, what may run, what must wait, what must be reopened, and when to stop.

It does not implement product functionality.
It does not audit as the Reviewer.
It does not invent decisions reserved in `PROJECT_GOAL.md`.

Autonomy by default. Human intervention by exception.

---

## 1. Authority chain

```text
PROJECT_GOAL.md
  ↓
GOALS_TREE.md
  ↓
ORCHESTRATOR.md          ← this file
  ↓
DECISION_AUTONOMY.md     ← DISCOVERY ask-gate (does not select work)
  ↓
EXECUTION_PROTOCOL.md    ← how the selected unit is built
  ↓
REVIEW_PROTOCOL.md       ← how the unit is audited
```

If documents conflict on scope, `PROJECT_GOAL.md` wins.
If documents conflict on inventory, `GOALS_TREE.md` wins.
If documents conflict on selection, this file wins.
If documents conflict on how to build, `EXECUTION_PROTOCOL.md` wins.
If documents conflict on how to ask the Human during DISCOVERY, `DECISION_AUTONOMY.md` wins.

---

## 2. Operating principle

```text
INSPECT REPOSITORY
  ↓
UNDERSTAND CURRENT STATE
  ↓
IDENTIFY NEXT OBJECTIVE INSIDE THE ACTIVE GOAL
  ↓
SELECT ONE ELIGIBLE UNIT
  ↓
EXECUTE  (EXECUTION_PROTOCOL)
  ↓
REVIEW   (REVIEW_PROTOCOL)
  ↓
ROUTE VERDICT
  ↓
UPDATE CONCEPTUAL STATE
  ↓
SELECT NEXT ELIGIBLE UNIT
  ↓
REPEAT
```

Do not ask “may I continue?” when eligibility, authority, and evidence are sufficient.

Do not start the next Task because the previous file was written.
Do not skip review.
Do not treat numeric Task order as a total order.

---

## 3. Modes

The same agent may play all modes in one session. Modes are sequential, not concurrent, for the same Task.

| Mode | Allowed | Forbidden |
|---|---|---|
| Orchestrator | discover, select, route, stop, reopen, report | edit product code, rewrite acceptance criteria |
| Builder | implement, test, repair, in-scope improve | approve own work, mark DONE |
| Reviewer | inspect, run checks, issue verdict | implement fixes, weaken criteria |

After Builder work, switch to Reviewer. Do not fix during review. If the verdict is `REJECTED`, switch back to Builder.

---

## 4. Conceptual state

Do not create a state database, extra reports directory, or control script.

Report state in the cycle output of this session. Persist only what already exists in the repository or what the current Task is explicitly allowed to change.

Task states (defined in `EXECUTION_PROTOCOL.md`):

```text
READY | IN_PROGRESS | VALIDATING | REVIEWING | IMPROVING | APPROVED | DONE | BLOCKED | CANCELLED
```

Phase states: `INACTIVE | ACTIVE | BLOCKED | DONE`
Goal states: `ACTIVE | BLOCKED | DONE`

`DONE` is reversible by reopen. `CANCELLED` is Human-only.

---

## 5. Discovery procedure

Every cycle, in this order:

1. Read `PROJECT_GOAL.md` (scope, constraints, reserved decisions, Goal DONE checklist, STOP)
2. Read `GOALS_TREE.md` (inventory, horizons, inferred dependencies)
3. List Phase files on disk: `.cursor/goals/phases/PHASE-*.md`
4. For the candidate Phase, list Task files: `.cursor/goals/tasks/PHASE-XX-name/TASK-*.md`
5. Inspect repository reality (layout, existing modules, tests, configs). Do not assume modules that are not present.
6. Reconstruct conceptual state from this session, prior cycle output in the conversation, and repository evidence. If unknown, treat the Task as not `DONE`.

If a name in memory is not on disk, it does not exist.

Discovery is relative to this repository’s workspace root (PETIA), not to any sibling project (including Bellia).

---

## 6. Horizon gate

The active Goal of this repository is PETIA (`PROJECT_GOAL.md`). Eligible horizons remain:

```text
eligible horizon = DISCOVERY ∪ BOOTSTRAP ∪ MVP
```

POST-MVP and FUTURE units are visible and must not be selected.

Exceptions (only these):

- the Human names the unit and authorizes the horizon
- a DISCOVERY/BOOTSTRAP/MVP Task has a hard dependency that is already `DONE` in a later horizon (rare; do not pull extra later-horizon work)

During DISCOVERY Tasks, follow `DECISION_AUTONOMY.md`. Do not stop to ask the Human for inferable or reversible product choices. DISCOVERY `DONE` does not authorize product code.

During MVP Tasks, apply the minimum safety and logging required by Goal constraints. That is not permission to execute later-horizon Phases.

Phase/Task files are on disk (PETIA instantiation 2026-08-29). Intended domain contract Phases are `PHASE-10` … `PHASE-14`. Product implementation is PHASE-14 only, after sufficient spec (PHASE-06), PHASE-13 architecture, Human-ratified stack, and Human authorization to write application code. Do not execute the superpowers implementation plan as the PHASE-14 script.

---

## 7. Eligibility

A Task is eligible when all of the following are true:

1. Its Phase exists on disk
2. Its file exists on disk
3. Its horizon is allowed (§6)
4. It is not `CANCELLED`
5. It is not currently `DONE`, unless it is being reopened
6. All hard dependencies are `DONE` (and not currently reopened)
7. All synchronization predecessors are `DONE`
8. No blocking dependency is active
9. No reserved Human Decision from `PROJECT_GOAL.md` is required to start this specific Task
10. No other Task is `IN_PROGRESS`, `VALIDATING`, `REVIEWING`, or `IMPROVING` in this session
11. The inferred or written contract is inside Goal In Scope, **or** the unit is BOOTSTRAP or DISCOVERY
12. Starting it would not modify protected files without authorization

A Task is not ineligible merely because:

- its markdown body is empty
- a later-numbered Task exists
- a soft dependency is unfinished
- the work looks difficult
- the Human has not approved the previous Task

### Empty body

If the Task/Phase file is empty, infer a minimal contract (`EXECUTION_PROTOCOL.md` §5.3). If inference requires a reserved Human Decision or would expand Out of Scope, the Task is not eligible → treat as `BLOCKED` only for that Task, and select a different eligible Task if one exists.

---

## 8. Selection algorithm

Choose exactly one Task.

1. Task explicitly named by the Human, if eligible
2. Task reopened in this cycle because of new blocking evidence
3. Task already `IN_PROGRESS` / `VALIDATING` / `REVIEWING` / `IMPROVING` in this session
4. Earliest blocked-unrelated eligible Task that unlocks the most downstream In-Scope work
5. Among remaining eligible Tasks, prefer unsatisfied hard-dependency producers
6. Tie-breaker: lowest Phase `XX`, then lowest Task `NN`

Never select randomly.
Never select POST-MVP/FUTURE under this Goal.
Never skip an eligible producer of a hard dependency in order to start its consumer.
Never treat DISCOVERY or BOOTSTRAP completion as permission to invent a stack or ship PHASE-14 code.
Never treat kit-legacy or Bellia clinic instantiation Tasks as permission to invent PETIA domain modules or a stack.
Never select a unit that is only listed as target inventory in `GOALS_TREE.md` and is not on disk.

If no Task is eligible and the Goal is not `DONE`:

- if at least one Task is `BLOCKED` on Human Intervention → stop, report the blocker
- if work remains but is all later-horizon → Goal In-Scope work may be complete; run Goal completion check
- if the only remaining Tasks have empty bodies that cannot be inferred without reserved decisions → Human Intervention
- otherwise report why nothing is eligible; do not invent work

---

## 9. Parallelism

Default in one agent session: one active Task.

Parallelizable means the Orchestrator may choose any eligible member of the set, not that it must run two implementations in the same context.

When reporting the next action, name:

- the selected Task
- other currently parallelizable eligible Tasks
- synchronization Tasks waiting on the set

Do not start a consumer Phase merely because a producer file exists. Required contracts must be `DONE`.

---

## 10. Invoke execution

When a Task is selected:

1. Output the cycle header (§17)
2. Set conceptual state `IN_PROGRESS`
3. Follow `EXECUTION_PROTOCOL.md` fully (inspect → plan → implement → test → review → improve → compare)
4. Do not mark `DONE` from Builder output alone
5. Route the Reviewer verdict (§11)

The Orchestrator must not bypass `REVIEW_PROTOCOL.md`.

---

## 11. Verdict routing

Valid review verdicts: `OBJECTIVE_PASS | APPROVED | REJECTED | BLOCKED`

### OBJECTIVE_PASS

The objective and acceptance criteria are evidenced, but the improvement pass has not yet run.

1. Do **not** mark the Task `DONE`
2. Transition `REVIEWING` → `IMPROVING`
3. Follow `EXECUTION_PROTOCOL.md` §6
4. If no in-scope improvement is implemented, promote to `APPROVED` without a second full review
5. If an improvement is implemented, test and review again

### APPROVED

Allowed only after the improvement pass has been completed or justifiably skipped (`EXECUTION_PROTOCOL.md` §6).

Then:

1. Confirm the review report actually matches `APPROVED` (`REVIEW_PROTOCOL.md` consistency rule)
2. `APPROVED` → `DONE`
3. Record handoff in the cycle output
4. Select the next eligible Task
5. Continue without asking the Human

### REJECTED

Autonomous rework. Not Human Intervention.

1. Keep the Issues
2. `REJECTED` → `IN_PROGRESS`
3. Builder corrects only the Issues plus necessary collateral
4. Test again
5. Review again
6. Repeat until `OBJECTIVE_PASS` (improvement not yet run) or `APPROVED` (improvement already run), or a limit / intervention condition

### BLOCKED

Human Intervention. Stop autonomous execution for this unit.

Do not convert a blocker into an assumption.
Do not continue adjacent Tasks that have a blocking dependency on this unit.
Other eligible Tasks without that blocking dependency may be selected if the blocker is local. If the blocker is Goal-wide (reserved decision needed by all remaining In-Scope paths), stop the Goal.

---

## 12. Reopen

`DONE` is not immutable.

Reopen a Task when new evidence shows:

- a later review found a defect in its contract
- a regression traces to it
- a downstream Task cannot satisfy its criteria because the producer is wrong
- the previous solution is inadequate
- an in-scope improvement cannot be made without changing it

```text
DONE → NEW EVIDENCE → IN_PROGRESS → TEST → REVIEW → (IMPROVE) → DONE
```

Reopen is normal engineering. It is not a protocol failure.

Limits: `EXECUTION_PROTOCOL.md` §12. Exceeding them → Human Intervention.

When reopening, consumers that hard-depend on the Task become ineligible until it is `DONE` again.

---

## 13. Continue vs stop

### Continue (autonomous)

- Builder finished an attempt
- tests failed with a diagnosable defect
- Reviewer returned `REJECTED`
- a relevant in-scope improvement was found
- Task became `DONE` and another eligible Task exists
- a Task was reopened and is still within limits
- empty Task body can be inferred inside scope

### Stop (Human Intervention)

Only when at least one is true:

- reserved Human Decision in `PROJECT_GOAL.md` is required and unset
- critical ambiguity that changes architecture, isolation boundaries, or external contracts
- conflict between `PROJECT_GOAL.md` and documented architecture that makes this Task impossible without violating one of them
- conflict between requirements that the agent cannot resolve from written authority
- risk of data loss or destructive schema change
- external breaking change
- unknown external dependency or missing credential required for a mandatory check
- environment required for critical tests is unavailable and cannot be substituted by an equivalent local/CI check
- protected file must change
- scope expansion is required to finish the Task
- iteration/reopen limits reached
- repository in an inconsistent state that autonomous repair cannot restore
- Goal/Phase would be declared `DONE` incorrectly without a missing Human Decision that the Goal itself requires

### Do not stop

- ordinary implementation choices
- DISCOVERY assumptions allowed by `DECISION_AUTONOMY.md`
- “How should this work?” when the agent can infer, apply a domain pattern, or assume reversibly
- refactor vs in-place fix inside scope
- missing nice-to-have tests that the Builder can add
- a hard Task that is still solvable
- “please confirm I should start the next Task”
- later-horizon ideas discovered during In-Scope work (record as `FUTURE_OPPORTUNITY` in the cycle output; do not implement; do not stop)

### Human Review (non-blocking)

The Human may later review Phase completion, Goal completion, architecture-relevant diffs, or important improvements.

Human Review is not a gate in the loop. Do not wait for it unless the Human asked to approve that class of change.

---

## 14. Recovery vs intervention

Failures are first handled by `EXECUTION_PROTOCOL.md` §11 (diagnose → repair → test → review).

The Orchestrator escalates to Human Intervention only after:

- autonomous repair is exhausted, or
- the failure class is one of the stop conditions in §13

```text
FAILURE
  ↓
SELF-DIAGNOSIS
  ↓
SELF-REPAIR
  ↓
TEST
  ↓
REVIEW
  ↓
SUCCESS → CONTINUE
RETRY ALLOWED → RETRY
CRITICAL / LIMIT / RESERVED DECISION → HUMAN INTERVENTION
```

---

## 15. Phase completion

A Phase may be `DONE` only when:

1. Every required Task in that Phase that belongs to the allowed horizon is `DONE` (exclude any completion exceptions written in `GOALS_TREE.md`)
2. Inferred or written Phase objective is met
3. No open Critical/High Issue remains against the Phase
4. Required integration evidence for that Phase exists
5. Reviewer has approved the Phase as a unit (`REVIEW_PROTOCOL.md` Phase review)
6. Downstream hard dependencies can consume its contracts

Do not infer Phase completion from “all Task files were touched”.

Empty Phase body: the Phase objective is the aggregate of its Task filenames plus the Goal mapping in `GOALS_TREE.md` §6.

POST-MVP Tasks inside a mixed Phase are not required for MVP Phase completion.

---

## 16. Goal completion

The Goal may be `DONE` only when every checkbox in `PROJECT_GOAL.md` Project DONE is satisfied with evidence.

If the Goal requires an end-to-end proof, that proof is a Goal-level condition. Do not skip it because no Phase is named “vertical integration”.

If a reserved Human Decision is still unset and a DONE checkbox depends on it, the Goal is `BLOCKED`, not `DONE`.

Do not declare Goal `DONE` because BOOTSTRAP Phases finished.
Do not declare Goal `DONE` because later-horizon Phases exist and were skipped.
Do not declare Goal `DONE` because files exist.

---

## 17. Cycle output

At cycle start:

```markdown
# Execution Cycle

- Goal:
- Horizon allowed:
- Phase:
- Task:
- Task state:
- Iteration:
- Hard dependencies:
- Soft dependencies:
- Parallelizable siblings:
- Human Intervention pending:
- Reserved decisions blocking this Task:
```

At cycle end:

```markdown
# Cycle Result

- Phase:
- Task:
- Iteration:
- Verdict:
- State transition:
- Evidence summary:
- Blocking Issues:
- In-scope improvements accepted:
- Out-of-scope opportunities recorded:
- Reopened Tasks:
- Next eligible Task:
- Parallelizable alternatives:
- Human Intervention required: YES | NO
- Continue autonomously: YES | NO
```

If Human Intervention is required, also emit:

```markdown
# Human Intervention Required

- Phase:
- Task:
- State: BLOCKED
- Decision or action required:
- Why the agent lacks authority or information:
- Options:
- Impact if each option is chosen:
- Work that may continue without this decision:
- Exact Human response needed:
```

Do not emit this template for ordinary implementation questions.

---

## 18. Task handoff

Create a handoff only after `APPROVED` and the improvement pass.

```markdown
# Task Handoff

- Phase:
- Task:
- Final state: DONE
- Contracts produced:
- Files changed:
- Validation evidence:
- Review evidence:
- Inherited constraints:
- Remaining LOW issues:
- FUTURE_OPPORTUNITY notes:
- Next eligible Task:
```

---

## 19. Protected control files

Product Tasks must not modify:

- `PROJECT_GOAL.md`
- `GOALS_TREE.md`
- `ORCHESTRATOR.md`
- `EXECUTION_PROTOCOL.md`
- `REVIEW_PROTOCOL.md`
- `DECISION_AUTONOMY.md`
- Phase definitions and Task acceptance criteria
- Canonical docs present in this workspace: `VISION.md`, `PRINCIPLES.md`, `ARCHITECTURE.md`, `DECISIONS.md`, `CONTRIBUTING.md`, `PETSHOP_OS_FEATURE_DISCOVERY.md`
- First-slice contract: `.cursor/docs/superpowers/specs/2026-08-29-petia-command-os-design.md`
- `.cursor/rules/*.mdc` when those files exist

unless the Human explicitly authorized that file, or the current Task’s written purpose is to change it.

`AGENTS.md` is not present in this workspace; do not invent it. If created later under Human authorization, treat it as protected once it exists.

The Orchestrator itself must not silently rewrite this protocol during product execution.

---

## 20. Success test for the Orchestrator

The Orchestrator is correct in a cycle only if it can answer:

1. What is the active Goal horizon?
2. Why this Task, not another?
3. Which hard dependencies are satisfied?
4. Why Human Intervention is or is not required?
5. What happens after the verdict?

If it cannot answer, do not start the Task.
