# Workflow

How the orchestrator drives the `database-expert` specialist. Read alongside `database-expert.md`.

## Overview

The orchestrator is the hub: every step returns to it before the next begins. It spawns the specialist as one of
three phases - planner, worker, reviewer - each a fresh, read-only subagent given only that phase's Input; phases
never call each other or share memory. The reviewer runs after the plan and after the work, returning a
`REVIEW_STATUS`; `REJECTED` sends the step back to be redone. Each step's per-review `Review Number` - a monotonic
counter the orchestrator increments before each review and never resets - caps it at two tries: a `REJECTED` still
standing at 2 halts the loop for manual intervention.

The task travels as a single `<slug>.md` file (see Appendices), the only state shared across phases. Each subagent
returns the full updated content rather than writing it; the orchestrator alone persists it - writing it after the
planner's first return, overwriting it after every worker/reviewer return.

A subagent never blocks on a decision it can reasonably make - it records anything it assumes, defers, or can't
settle as a `<CRITICALITY>`-tagged Assumption, Risk, or Gap in `<slug>.md` and keeps going, for the human to resolve
at the end of phase 3.

## Phase 1 - Plan

1. Orchestrator -> Planner - returns the initial `<slug>.md`: Requirements/Constraints, the plan as tasks (each its
   own `### Task N` with `TASK_STATUS TO_DO` and a `VERIFY_STATUS`), and Risks/Gaps/Assumptions. Orchestrator writes
   the file.
2. Orchestrator -> Reviewer - reviews the plan, returns plan `REVIEW_STATUS` plus findings, merged into `<slug>.md`
   under Reviews.

On `REJECTED`, re-spawn the Planner with the findings and repeat phase 1 while the plan `Review Number` is `< 2`.
Advance to phase 2 once `APPROVED` or `PARTIALLY_APPROVED`; else halt (see Overview).

## Phase 2 - Work

1. Orchestrator -> Worker - implements the approved plan (change/rollback/verify SQL) and returns `<slug>.md` with
   each task's `TASK_STATUS` advanced to `IN_PROGRESS` then `DONE`, its `VERIFY_STATUS` corrected if needed, and
   Risks/Gaps/Assumptions revised. Orchestrator overwrites the file.
2. Orchestrator executes - the specialist is read-only, so the orchestrator runs exactly what each task's
   `VERIFY_STATUS` calls for (see Appendices) on a copy, then appends the raw output to `<slug>.md` as evidence,
   noting the copy's affected-table row counts - lock, timing, and `EXPLAIN` evidence only holds on a production-like
   copy. A `NONE` task is skipped; any other task with no evidence appended is a gap, not a skip.
3. Orchestrator -> Reviewer - reviews the finished work and the step-2 evidence against the plan and acceptance
   criteria; if the plan review was `PARTIALLY_APPROVED`, also re-check its MEDIUM findings were addressed or are
   still consciously accepted. Returns work `REVIEW_STATUS` plus findings, merged into `<slug>.md`.

A Reviewer finding required evidence missing or inconclusive returns `REJECTED` naming what to (re-)run - back to
step 2, not the Worker, then re-review. On a Worker-content `REJECTED`, re-spawn the Worker and repeat phase 2 while
the work `Review Number` is `< 2`. If a finding shows the plan itself is wrong, don't loop back to phase 1 - record it
as a `CRITICAL` Gap and halt for manual intervention. Advance to phase 3 on `APPROVED` or `PARTIALLY_APPROVED`; else
halt (see Overview).

## Phase 3 - Human sign-off

Reached once the work review is `APPROVED` or `PARTIALLY_APPROVED`. Human sign-off is required before applying when
any holds:

- The plan contains a destructive step (see `database-expert.md` Database Guardrails) or a flagged execution hazard.
- The work review is `PARTIALLY_APPROVED` - a human accepts or clears its conditions first.
- `<slug>.md` still carries a `CRITICAL`- or `HIGH`-tagged Risk, Gap, or Assumption.

The human resolves outstanding `CRITICAL`/`HIGH` items, highest first, then signs off. `MEDIUM`/`LOW` items don't
block sign-off; they travel with `<slug>.md` as a record. Otherwise the orchestrator delivers the ready-to-run SQL
and apply steps directly.

## Appendices

### `<slug>.md`

The single artifact the task travels in. Every phase is read-only and returns the file's content; only the
orchestrator persists it. Skeleton: `templates/slug.md` - Reviews (plan + work, each a `Review Number`, a
`REVIEW_STATUS`, plus `<CRITICALITY>` findings), Requirements, Constraints, Tasks (each with `TASK_STATUS` and
`VERIFY_STATUS`), and Risks/Gaps/Assumptions (each `<CRITICALITY>`).

### TASK_STATUS

`TO_DO` (not started) -> `IN_PROGRESS` (being worked on) -> `DONE` (complete and verified).

### VERIFY_STATUS

- `ROLLBACK_AND_EXPLAIN` - changes schema/data and makes a performance claim (e.g. a new index): run the
  up -> verify -> down -> up cycle and the named `EXPLAIN (ANALYZE, BUFFERS)`.
- `ROLLBACK_ONLY` - changes schema/data with no performance claim (e.g. add a column, backfill, add a constraint):
  run only the up -> verify -> down -> up cycle.
- `EXPLAIN_ONLY` - no schema/data change but makes a performance claim (e.g. a query rewrite for speed): run only
  the named `EXPLAIN`.
- `NONE` - neither applies (correctness-only fix, pure analysis/advice): execute nothing.

A destructive task is still tagged accordingly, but its up -> down -> up cycle only proves the steps are reversible,
not that dropped data is recoverable - the Database Guardrails' phased-rollout rule protects against data loss, and
the phase 3 approval gate still applies regardless of tag.

### CRITICALITY

`CRITICAL` (blocks progress, fix immediately) > `HIGH` (fix before proceeding) > `MEDIUM` (fix in the current phase)
> `LOW` (can defer).

### REVIEW_STATUS

- `APPROVED` - no issues or only LOW issues. Ready to proceed.
- `PARTIALLY_APPROVED` - no CRITICAL/HIGH; has MEDIUM that don't block. Proceed with conditions.
- `REJECTED` - has CRITICAL or HIGH issues. Back to the previous step.
