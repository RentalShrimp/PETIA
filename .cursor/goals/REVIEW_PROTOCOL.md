# Review Protocol

Version: 1.0.0
Status: Operational
Review date: 2026-08-29

The Reviewer must independently try to prove that the current Task is **not** `DONE`.

Product: **PETIA** (Pet Shop Operating System). Audit against this Goal and the Command OS first-slice spec, not against sibling clinic products.

Review is a mandatory stage of the execution loop. It is not optional, not a courtesy, and not a Human gate.

Approval requires evidence. Intention, code volume, confidence, and visual inspection are not enough.

---

## 1. Authority

This file owns: how to audit, Issue format, severity, verdicts, improvement accept/reject, Phase-as-unit review.

It does not select work (`ORCHESTRATOR.md`).
It does not implement (`EXECUTION_PROTOCOL.md`).
It does not change `PROJECT_GOAL.md`.

On review questions, this file wins.

---

## 2. Reviewer mode

The Reviewer must not:

- implement functionality
- fix code during the review pass
- rewrite tests to hide failures
- modify acceptance criteria
- change project scope
- approve incomplete work
- ignore failures because they look unrelated
- make reserved Human Decisions

The Reviewer may: run commands, inspect files, run tests, read diffs, produce Issues and a verdict.

If the same agent just acted as Builder, it must switch mode: re-read the contract, re-inspect the diff, and attempt to falsify completion. Do not rubber-stamp the Builder handoff.

The Builder handoff is input, not proof.

A checked box inside a Markdown file is not evidence of review approval.

---

## 3. Review order

1. Task objective (written or inferred)
2. Acceptance criteria
3. Out-of-scope / horizon rules
4. Allowed vs protected files
5. Implementation diff
6. Tests and whether they actually lock the behavior
7. Validation evidence
8. Architecture and integration
9. Security, isolation, data integrity
10. Regressions
11. Improvement pass and before/after comparison
12. Verdict

---

## 4. Required review areas

Every area must be considered. Use `NOT APPLICABLE` only with a reason.

### 4.1 Functional correctness

- Was the expected behavior implemented?
- Were the requirements met?
- Does the final state match the Task objective?

### 4.2 Tests

- Do relevant tests exist?
- Do they validate the real behavior, not only mocks?
- Would they fail if the behavior broke?
- Are important edge cases missing?
- If tests are missing and the behavior is testable, that is an Issue (usually Medium or High)

### 4.3 Regression

- Did previously valid behavior break?
- Did relevant existing tests fail?
- A failure may be accepted only with evidence that the old behavior was wrong and the Task contract required the change

### 4.4 Architecture

- Are module boundaries respected?
- Was unnecessary coupling introduced?
- Were responsibilities placed in the wrong layer?
- Does the handler / application layer named by the Goal remain the transactional authority?
- Do events replace persistence? (must not; timeline is a projection)
- Does the LLM compose SQL, execute privileged queries, or decide reserved outcomes? (must not)
- Are isolation boundaries named by the Goal (`tenant_id`, `unidade_id`) propagated at the edges that implement them?

Use the repository’s ratified architecture and ADRs that exist on disk. In this workspace: `DECISIONS.md` holds decisions when they are written; `ARCHITECTURE.md` is a **stub**, not Accepted. The Command OS first-slice spec **is** the design contract for this wave. Reject violations against that spec and constraints in `PROJECT_GOAL.md`. Do not invent a stack. If a Task would require an unset leftover (language/framework/DB, messaging vendor, PSP, first code slice), that is `BLOCKED` or `NOT APPLICABLE` with that reason — not a license to pick a vendor or copy Bellia DEC-010.

### 4.5 Maintainability

- Is the change understandable?
- Is it sustainable?
- Is there unnecessary duplication or complexity?

### 4.6 Security

- New vulnerabilities?
- Secrets committed?
- Sensitive data in logs?
- Authorization only on the client?

### 4.7 Performance

- Relevant regression?
- Justified improvement opportunity related to this Task?

Premature optimization is not a required finding. Unjustified heavy mechanisms are a finding.

### 4.8 Data integrity

- Can data be lost or corrupted?
- Are migrations safe?
- Destructive operations without authorization?

### 4.9 Scope

- Did the agent change anything outside the objective?
- Did it implement POST-MVP/FUTURE work under an In-Scope Task?
- Did it weaken criteria?
- Did it modify protected files without authorization?

### 4.10 Improvement

- Was the improvement pass performed (`EXECUTION_PROTOCOL.md` §6)?
- Are implemented improvements relevant, bounded, and verified?
- Is the final state actually better than the initial state (`EXECUTION_PROTOCOL.md` §7)?

### 4.11 Evidence

- Does each mandatory criterion have evidence?
- Is any criterion only “looks correct”?

### 4.12 Boundary isolation (when the Goal names an isolation boundary)

- The isolation context cannot be omitted
- Cross-boundary access is impossible through the changed path
- Identifiers cannot bypass isolation filters
- Isolation is proven by tests when the Task touches persistence or API

If the Goal does not name an isolation boundary, mark `NOT APPLICABLE` with that reason.

### 4.13 Integration (when the Task spans layers)

Default product path after architecture is ratified (Command OS):

```text
Tela da loja  ──┐
                ├──► Comando ──► Orquestrador ──► Handler ──► SQL parametrizado
WhatsApp+LLM ──┘                                         │
                                                         ▼
                                                   Evento + ComandoLog
                                                         │
                                                         ▼
                                                   Timeline (projeção)
```

While no application layers exist on disk, mark layered integration `NOT APPLICABLE` with that reason unless the Task implements the authorized slice. After PHASE-14 starts, judge against the first-slice spec and any Accepted `ARCHITECTURE.md`: writes go through named commands; reads do not go through the agent; the frontend is never the tenancy/authorization authority. The LLM is a client of the catalog, not the author of the database.

The feature must work through the layers it claims to connect. Module existence is not integration evidence.

---

## 4.14 Repository bind — commands the Reviewer MUST attempt

Attempt only what exists. At bind time this workspace has **no** install/build/lint/typecheck/test scripts and **no** CI workflows.

| Check | Action |
|---|---|
| Lint / typecheck / unit / integration tests | If scripts appear later (`package.json`, `pyproject.toml`, Makefile, etc.), run the real commands. If absent → record `NOT PRESENT`; do not invent. |
| Build | Same rule. |
| Docs / decision consistency | Inspect against existing canonical docs: `VISION.md`, `PRINCIPLES.md`, `ARCHITECTURE.md`, `DECISIONS.md`, `CONTRIBUTING.md`, `PETSHOP_OS_FEATURE_DISCOVERY.md`, and `.cursor/docs/superpowers/specs/2026-08-29-petia-command-os-design.md` |
| Protected files | Same factual list as `ORCHESTRATOR.md` §19 |

A Markdown checkbox is never evidence that these checks ran.

---

## 5. Issue format

### Issue N

- `SEVERITY`: CRITICAL | HIGH | MEDIUM | LOW
- `FILE`: path
- `LINE`: line or range, or `n/a`
- `PROBLEM`: precise description
- `EXPECTED`: required behavior
- `EVIDENCE`: command, test, or inspection
- `RECOMMENDATION`: correction
- `BLOCKING`: YES | NO

Issues must be concrete and reproducible.

---

## 6. Severity

### CRITICAL — always blocking — verdict cannot be APPROVED

Examples: security vulnerability; isolation breach; data loss; production-critical flow broken; mandatory criterion not implemented; destructive unprotected change.

### HIGH — always blocking

Examples: major functional failure; broken API contract; migration failure; missing required integration; unreliable transactional behavior; LLM given privileged authority or allowed to compose SQL.

### MEDIUM — blocking unless the Task contract explicitly defers it

Examples: missing required tests for new behavior; meaningful regression risk; incomplete error handling for a required path; improvement pass skipped without justification.

### LOW — not automatically blocking

Examples: minor naming; non-critical cleanup; docs polish.

LOW Issues must still be recorded. They may be left as inherited constraints in the handoff.

---

## 7. Verdicts

Exactly one per review pass: `OBJECTIVE_PASS | APPROVED | REJECTED | BLOCKED`

### OBJECTIVE_PASS

Use after the first review of an implementation attempt, when:

- every mandatory acceptance criterion is `PASS`
- required evidence exists
- no unresolved Critical, High, or blocking Medium Issue
- relevant tests pass
- scope respected
- the improvement pass has **not** yet run in this attempt

`OBJECTIVE_PASS` is not Task completion. The Orchestrator must enter `IMPROVING`.

Do not return `OBJECTIVE_PASS` after the improvement pass has already run in this attempt.
Do not return `APPROVED` before the improvement pass has run or been justifiably skipped.

### APPROVED

Use only after the improvement pass has run or been justifiably skipped. All of:

- every mandatory acceptance criterion is `PASS`
- required evidence exists
- no unresolved Critical, High, or blocking Medium Issue
- relevant tests pass
- scope respected
- no unauthorized protected-file change
- improvement pass done or justifiably skipped
- accepted improvements compared and kept only if better
- no reserved Human Decision left implicit

If the header says `APPROVED` but blocking Issues or missing evidence remain, the Orchestrator must treat the Task as `REJECTED` or `BLOCKED`. The Reviewer must not emit that inconsistent header.

### REJECTED

Any of:

- mandatory criterion FAIL or NOT VERIFIED
- insufficient evidence for a mandatory criterion
- blocking Issue present
- required tests fail
- architecture violates the contract or Goal constraints
- isolation not proven when required
- implementation incomplete
- improvement made the system worse and was not reverted
- scope expanded

`REJECTED` means autonomous rework. It is not Human Intervention.

### BLOCKED

Only when the Reviewer cannot determine correctness or cannot continue without authority the agent lacks:

- reserved Human Decision required
- mandatory external service/credential unavailable
- repository lacks information needed to judge correctness
- Task conflicts with an approved architectural decision and with the Goal (unresolvable)
- iteration/reopen limit reached
- data-loss or destructive-change risk requiring Human authority

Do not use `BLOCKED` for ordinary defects. Those are `REJECTED`.

Do not use `BLOCKED` to ask permission to continue a valid next step.

---

## 8. Criterion status

Each acceptance criterion:

| Status | Meaning |
|---|---|
| PASS | evidenced as satisfied |
| FAIL | evidenced as not satisfied |
| NOT VERIFIED | no sufficient evidence |
| NOT APPLICABLE | not relevant, with justification |

`NOT VERIFIED` on a mandatory criterion prevents `APPROVED`.

---

## 9. Improvement accept / reject

When the Builder implemented an improvement, the Reviewer must compare:

```text
CURRENT STATE vs NEW STATE
```

Accept the improvement only if `NEW STATE > CURRENT STATE` using `EXECUTION_PROTOCOL.md` §7.

Reject (Issue, usually Medium or High) when:

- the improvement is out of scope
- benefit is not evidenced
- complexity increased without benefit
- valid behavior regressed
- architecture or security worsened

If comparison fails, require revert. Do not approve a worse system because the original objective still passes.

Out-of-scope ideas listed as `FUTURE_OPPORTUNITY` are not Issues unless the Builder implemented them.

---

## 10. Reopen recommendation

The Reviewer may recommend reopening a previously `DONE` Task when this review’s evidence shows that Task’s contract is now false.

State:

- which Task to reopen
- the new evidence
- whether it is a blocking dependency for the current Task

The Orchestrator performs the reopen (`ORCHESTRATOR.md` §12). The Reviewer does not edit that Task during this pass.

---

## 11. Phase review

When the Orchestrator requests Phase completion review, audit the Phase as a unit:

- required horizon Tasks are `DONE`
- Phase objective (written or inferred) is met
- contracts needed by downstream Phases exist
- no open Critical/High Issue
- integration evidence for the Phase exists

Do not approve a Phase because every Task file received some edit.

Goal completion review follows `PROJECT_GOAL.md` Project DONE and `ORCHESTRATOR.md` §16.

Human Review of a Phase or Goal is optional and non-blocking unless the Human required it.

---

## 12. Review report template

```markdown
# Review Report

- Task:
- Phase:
- Iteration:
- Date:
- Verdict:

## Acceptance Criteria

| Criterion | Status | Evidence |
|---|---|---|
| | PASS/FAIL/NOT VERIFIED/NOT APPLICABLE | |

## Validation Performed

- Command:
- Result:
- Limitations:

## Issues

[Each Issue in §5 format]

## Scope Review

- Allowed files respected:
- Protected files respected:
- Unrelated / out-of-horizon changes:

## Architecture / Security / Data

- Architecture:
- Security:
- Isolation:
- Data integrity:

## Regression Review

- Tests executed:
- Result:
- Known limitations:

## Improvement Review

- Discovery performed:
- Candidates rejected (reason):
- Improvements accepted:
- Before/after comparison:
- FUTURE_OPPORTUNITY left unimplemented:

## Reopen Recommendations

- None | Task id + evidence

## Final Reasoning

[Why OBJECTIVE_PASS, APPROVED, REJECTED, or BLOCKED]
```

---

## 13. Success test for review

A review is valid only if:

1. it could have produced `REJECTED`
2. every mandatory criterion has a status and evidence pointer
3. the verdict matches the Issues
4. `BLOCKED` is used only for intervention-class conditions
5. `OBJECTIVE_PASS` is never treated as Task `DONE`
6. `APPROVED` means the improvement pass ran, the system is not worse than before, and the objective is evidenced
