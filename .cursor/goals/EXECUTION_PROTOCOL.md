# Execution Protocol

Version: 1.0.0
Status: Operational
Review date: 2026-08-29 

This file defines how a selected Task is executed.

Product: **PETIA** (Pet Shop Operating System). Bind to this workspace, not to sibling products.

It does not select the Task. The Orchestrator does that.
It does not own verdict taxonomy details. The Review Protocol does that.

A Task is a goal plus acceptance criteria, not a literal script.
Writing code is not completion.
Passing existing tests is not completion.

Autonomy by default. Human intervention by exception.

---

## 1. Purpose

Execute one selected unit until it is justifiably `DONE`, `BLOCKED`, or `CANCELLED`.

Required loop:

```text
INSPECT
  ↓
UNDERSTAND CURRENT STATE
  ↓
PLAN
  ↓
IMPLEMENT
  ↓
TEST
  ↓
VALIDATE
  ↓
REVIEW
  ↓
OBJECTIVE_PASS? ──NO──→ REWORK → TEST → REVIEW
  │
 YES
  ↓
SEARCH RELEVANT IMPROVEMENTS
  ↓
FOUND? ──NO──→ APPROVED → DONE
  │
 YES
  ↓
IMPLEMENT IMPROVEMENT
  ↓
TEST
  ↓
REVIEW
  ↓
COMPARE BEFORE / AFTER
  ↓
NEW STATE > OLD STATE? ──NO──→ REVERT OR REWORK
  │
 YES
  ↓
ACCEPT → UPDATE STATE → DONE
```

Forbidden loop:

```text
IMPLEMENT → DONE
```

---

## 2. Authority

Follow, in this order:

1. `PROJECT_GOAL.md`
2. Selected Phase and Task (written contract, or inferred if empty)
3. `ORCHESTRATOR.md` (eligibility already decided)
4. `DECISION_AUTONOMY.md` when the unit is DISCOVERY (ask-gate)
5. This file
6. `REVIEW_PROTOCOL.md`
7. Repository architecture, ADRs, and `.cursor/rules` when those files exist

Do not expand Out of Scope from `PROJECT_GOAL.md` (BOOTSTRAP work is allowed because it enables Goal execution; it is not PETIA domain scope expansion).
Do not implement POST-MVP or FUTURE work during an In-Scope Task.
Do not modify `PROJECT_GOAL.md` during product Task execution (instantiation-bind authorization is session-scoped and does not carry into ordinary cycles).

For **DISCOVERY** horizon Tasks: `IMPLEMENT` means writing the named artifact under `.cursor/docs/product-discovery/` (and assumptions). Creating application code, UI, APIs, migrations, or choosing a stack is a STOP condition. Follow `DECISION_AUTONOMY.md` before any Human question.

---

## 2.1 Repository bind (PETIA workspace)

Validation commands must come from files on disk. Inventory at bind time:

| Concern | On disk? | Command / note |
|---|---|---|
| Package manifest | NOT PRESENT | No `package.json`, `pnpm-lock.yaml`, `pyproject.toml`, `go.mod`, or `Cargo.toml` |
| Install | NOT PRESENT | — |
| Build | NOT PRESENT | — |
| Lint | NOT PRESENT | No project lint script / Makefile target |
| Typecheck | NOT PRESENT | — |
| Test | NOT PRESENT | No `npm test`, `pytest`, `go test`, etc. |
| CI workflows | NOT PRESENT | No `.github/workflows/` |
| Application layout | NOT PRESENT | No `apps/`, `src/`, or `packages/` |
| Architecture spec | STUB | `ARCHITECTURE.md` exists but is not Accepted. First-slice contract: `.cursor/docs/superpowers/specs/2026-08-29-petia-command-os-design.md`. Do not invent a stack. The implementation plan’s Python/FastAPI/Postgres proposal is not ratification. Toolchain remains NOT PRESENT until manifests exist |
| Phase / Task tree | PRESENT | `.cursor/goals/phases/`, `.cursor/goals/tasks/`. Discover from disk. Do not invent extra units. Do not reuse Bellia clinic inventory. Do not execute `.cursor/docs/superpowers/plans/2026-08-29-petia-command-os.md` as PHASE-14. |

If a command or pipeline is **not present** on disk, the Builder records `NOT PRESENT` in evidence and does **not** invent an install/build/lint/test pipeline.

When manifests and scripts appear later, use those real scripts as the bind source of truth and update this section in an authorized control-file change.

Relevant existing checks for docs-only Tasks: file inspection, diff review, and consistency against `VISION.md`, `PRINCIPLES.md`, `DECISIONS.md`, `ARCHITECTURE.md`, `CONTRIBUTING.md`, `PETSHOP_OS_FEATURE_DISCOVERY.md`, and the first-slice spec.

---

## 3. Roles as modes

Same agent, exclusive modes (`ORCHESTRATOR.md` §3).

### Builder

- inspect, plan, implement, test, repair, apply accepted in-scope improvements
- record evidence
- respond to Reviewer Issues
- must not mark the Task `APPROVED` or `DONE`

### Reviewer

- follow `REVIEW_PROTOCOL.md`
- must not implement during the review pass

### Human

- reserved decisions in `PROJECT_GOAL.md`
- Human Intervention stop conditions in `ORCHESTRATOR.md` §13
- optional Human Review after the fact

The Builder must not invent reserved decisions.

---

## 4. State machine

Valid states:

```text
READY
IN_PROGRESS
VALIDATING
REVIEWING
IMPROVING
APPROVED
DONE
BLOCKED
CANCELLED
```

Valid transitions:

```text
READY → IN_PROGRESS
IN_PROGRESS → VALIDATING
IN_PROGRESS → BLOCKED             # reserved decision or stop condition found
VALIDATING → REVIEWING
VALIDATING → IN_PROGRESS          # diagnosable test failure, autonomous repair
VALIDATING → BLOCKED              # mandatory check impossible
REVIEWING → IN_PROGRESS           # REJECTED → rework
REVIEWING → IMPROVING             # OBJECTIVE_PASS
REVIEWING → APPROVED              # final review after improvement pass
REVIEWING → BLOCKED
IMPROVING → VALIDATING            # improvement implemented
IMPROVING → APPROVED              # no relevant improvement implemented
IMPROVING → IN_PROGRESS           # improvement failed comparison or review
IMPROVING → BLOCKED
APPROVED → DONE
DONE → IN_PROGRESS                # reopen
BLOCKED → IN_PROGRESS             # after Human resolution
any non-DONE state → CANCELLED    # Human only
```

Do not force an invalid transition to look complete.

`CANCELLED` is Human-only.

Illegal (not expressible): `IN_PROGRESS → DONE`, `IN_PROGRESS → APPROVED`, `VALIDATING → DONE`.

---

## 5. Standard Task cycle

### 5.1 Inspect

Read:

- `PROJECT_GOAL.md`
- current Phase and Task files (if they exist)
- first-slice spec and canonical docs when the Task consumes them
- applicable architecture, ADRs, rules
- current repository state for the affected area
- previous Issues and evidence in this session

Inspect means search, open, and verify. Do not rely on memory of modules that may not exist.

### 5.2 Understand current state

Record, at least:

- what already exists
- what is missing for the Task objective
- relevant tests
- relevant risks (data, isolation, security)
- constraints that must not be violated

### 5.3 Infer or extract the contract

If the Task file contains Goal, scope, and acceptance criteria, use them.

If the Task file is empty, infer a **minimal** contract:

| Field | Source |
|---|---|
| Objective | Task filename + Phase filename + Goal mapping |
| In scope | Smallest change that achieves the objective |
| Out of scope | Goal Out of Scope, other Phases, POST-MVP/FUTURE |
| Acceptance criteria | Verifiable statements derived from the objective |
| Evidence | Tests, build, inspection, diffs appropriate to the change |

Write the inferred contract in the Builder handoff. Do not write it into the Task file unless this Task’s purpose is to author that file.

If inference requires a reserved Human Decision, stop: `BLOCKED`.

### 5.4 Plan

Produce a short plan before editing:

- current state
- proposed change
- expected benefit
- files likely touched
- tests to run
- risks
- what will not be changed

Prefer the smallest complete change that satisfies the objective.

### 5.5 Implement

- change only what the contract requires, plus technically necessary collateral
- do not add unrelated features
- do not perform cosmetic refactors as the delivery
- keep privileged / transactional rules in the handler / application layer the Goal names; the LLM must not compose SQL, skip command confirmation, or execute those rules by itself
- do not assume nonexistent modules

### 5.6 Test and validate

Set state `VALIDATING`.

Run the relevant checks that exist in the repository. Typical evidence sources:

- build
- unit / integration / isolation tests
- static checks
- targeted inspection (search, diff, schema, logs)
- manual verification only when no automated equivalent exists

If a critical check cannot run because the environment is unavailable, attempt an equivalent local/CI substitute. If none exists and the criterion is mandatory → `BLOCKED` (`ORCHESTRATOR.md` §13).

Test failures that are diagnosable defects → autonomous repair (`§11`), not Human Intervention.

Record evidence (`§10`). “It works” is not evidence.

### 5.7 Review

Set state `REVIEWING`.

Follow `REVIEW_PROTOCOL.md` in Reviewer mode.

- If the improvement pass has not run in this attempt, a successful review is `OBJECTIVE_PASS`, never Task `DONE`.
- If the improvement pass has run or was justifiably skipped, a successful review is `APPROVED`.

Route the verdict through `ORCHESTRATOR.md` §11.

### 5.8 Rework

On `REJECTED`:

1. State `IN_PROGRESS`
2. Address every blocking Issue
3. Test
4. Review again
5. Count iterations (`§12`)

The Builder may revert the attempt and re-implement if the review shows the approach is wrong.

---

## 6. Self-improving loop

After the objective review passes, state `IMPROVING`.

This pass is mandatory after `OBJECTIVE_PASS`. Skipping discovery is allowed only when all are true:

- the Task produces no runtime behavior (inventory, structure, or protocol text only)
- no related code/tests/config exist to improve
- the reason is recorded

After discovery, if budget remains (`§12`) and a candidate was just accepted, one more discovery is allowed. When the budget is exhausted, record leftover ideas as `FUTURE_OPPORTUNITY` and proceed to `APPROVED`. Do not start a new discovery pass after the final review that produced `APPROVED`.

### 6.1 Discover

Search for problems and improvements **related to this Task, this Phase, and the Goal**:

- bugs
- inconsistencies
- duplication
- architecture violations
- security issues
- reliability issues
- missing tests for the behavior just added
- unjustified complexity
- automation that removes a fragile manual step

Do not search the whole repository for generic cleanups.

### 6.2 Filter

Implement an improvement only when all are true:

- related to the Goal, Phase, or current Task
- technically justified
- verifiable
- measurable when reasonably possible
- compatible with architecture and constraints
- proportional to benefit
- free of unnecessary scope

Principle:

```text
Improve the system only when the improvement is relevant, justified, bounded and verifiable.
```

Reject:

- overengineering
- cosmetic refactors
- unnecessary abstractions
- gratuitous architectural change
- features outside the Task/Goal
- personal style-only changes
- premature optimization
- complexity without proven benefit

Out-of-scope discoveries: record as `FUTURE_OPPORTUNITY` in the cycle output. Do not implement. Do not create files for them.

### 6.3 Apply, test, review, compare

For each accepted candidate (within `§12` limits):

1. Snapshot the relevant current state (behavior, tests, complexity)
2. Implement
3. Test
4. Review
5. Compare before / after (`§7`)
6. Keep only if `NEW STATE > CURRENT STATE`
7. Otherwise revert and record why

If no candidate passes the filter, transition `IMPROVING` → `APPROVED`.

---

## 7. Before / after comparison

Every improvement, and any change that replaces a working approach, must be judged as:

```text
CURRENT STATE
  → PROPOSED CHANGE
  → EXPECTED BENEFIT
  → IMPLEMENTATION
  → VALIDATION
  → COMPARISON
  → BETTER STATE or REJECT
```

`NEW STATE > CURRENT STATE` considering, in order:

1. Goal adherence
2. Functional correctness
3. Security and isolation boundaries named by the Goal
4. Data integrity
5. Reliability
6. Maintainability
7. Performance (only if measured or obviously relevant)
8. Complexity (lower is better when other dimensions are not worse)

A change that works but does not improve the system on these dimensions must be reverted.

Regression of valid existing behavior is a failed comparison unless the Task’s written contract requires the behavior change and tests prove it.

---

## 8. Completion criteria

A Task may become `DONE` only when all are true:

1. Objective achieved
2. Acceptance criteria satisfied (written or inferred)
3. Implementation validated with evidence
4. Relevant tests executed
5. No known regressions
6. Review performed and `APPROVED`
7. Improvement pass performed or justifiably skipped
8. Accepted improvements implemented, tested, reviewed, and compared
9. Scope respected
10. Protected files untouched without authorization
11. No unresolved Critical, High, or blocking Medium Issue
12. Handoff recorded in the cycle output

Not sufficient:

- code exists
- Builder is confident
- existing tests happened to pass
- happy path only
- review skipped
- improvement pass skipped without justification
- a Markdown checkbox was ticked

---

## 9. Scope and protected files

Do not modify files outside the Task scope unless:

1. the change is technically required for the objective
2. the file is not protected
3. the reason is documented in the Builder handoff
4. the Reviewer verifies it

Protected files: `ORCHESTRATOR.md` §19.

If a protected file must change and the Task is not authorized to change it → `BLOCKED`.

Do not weaken acceptance criteria.
Do not edit Phase/Task files as a side effect of implementing product behavior.

---

## 10. Evidence

Every `DONE` Task needs evidence for:

- each acceptance criterion
- automated tests that exist and are relevant
- build/static checks when the stack has them
- persistence/migration validation when persistence changed
- API/integration validation when contracts changed
- isolation when isolation boundaries changed
- no known regressions
- scope and protected-file compliance

Each evidence item must include:

- command or inspection performed
- result
- relevant output
- files involved
- limitations or skipped checks, with reason

If the repository has no evidence directory, do not create one. Put evidence in the cycle output.

Looks correct ≠ evidence.

---

## 11. Autonomous recovery

On failure:

1. Identify
2. Classify
3. Diagnose
4. Attempt repair
5. Test
6. Review

Do not request a Human on the first failure.

| Class | Action |
|---|---|
| Defect in this change | repair, retest |
| Broken test that should fail | fix product or fix invalid test, with evidence |
| Transient command failure | retry once, then diagnose |
| Environment missing for optional check | skip with recorded limitation if criterion remains proven another way |
| Environment missing for mandatory check | `BLOCKED` |
| Reserved decision required | `BLOCKED` |
| Limit reached | `BLOCKED` |

```text
FAILURE → SELF-DIAGNOSIS → SELF-REPAIR → TEST → REVIEW
  SUCCESS → CONTINUE
  RETRY ALLOWED → RETRY
  CRITICAL / LIMIT → HUMAN INTERVENTION
```

---

## 12. Limits (anti-infinite-loop)

Defaults per Task:

| Limit | Value |
|---|---|
| Implementation / review cycles | 5 |
| Consecutive validation failures with no new diagnosis | 3 |
| Unchanged rejection cycles | 2 |
| Improvement implementations after first objective pass | 2 |
| Reopens per Task per Phase | 3 |

Unchanged rejection means the same blocking Issue remains, or files did not materially change, or evidence did not improve.

When a limit is reached:

```text
Task → BLOCKED
```

Report: Task, Phase, iteration, limit, unresolved Issues, attempted fixes, evidence, decision needed.

Do not polish forever.
Do not reopen forever.
Do not implement a rejected improvement a second time in the same Task.

---

## 13. Builder handoff format

```markdown
# Builder Handoff

- Phase:
- Task:
- Attempt:
- Inferred contract used: YES | NO
- Objective:
- Current state summary:
- Proposed change:
- Expected benefit:
- Files changed:
- Acceptance criteria self-check:
- Validation commands and results:
- Improvement candidates considered:
- Improvements implemented:
- Before/after comparison:
- FUTURE_OPPORTUNITY:
- Known limitations:
- Scope exceptions:
- Human Intervention required: YES | NO
```

---

## 14. Architectural constraints during implementation

From `PROJECT_GOAL.md`, always:

- the persistence source of truth is whatever the Goal names; do not invent a second
- named commands are the write path; handlers own transactional / privileged rules
- events do not replace persistence; the pet timeline is a projection of events
- isolation boundaries named by the Goal (`tenant_id`, `unidade_id`) are enforced at the edges that implement them
- the LLM does not compose SQL, does not execute privileged queries, and does not decide reserved outcomes by itself
- the repository must reflect reality
- do not add Out of Scope functionality (including Bellia clinic leftovers and the 22-domain discovery dump)

If the Task would require violating a ratified architectural decision in order to finish, `BLOCKED` for Human Intervention.

---

## 15. Success test for execution

Execution is correct only if another agent can see, from the handoff and review:

1. what state was required
2. what changed
3. why it is better than before
4. which criteria passed, with evidence
5. what was not done, on purpose
