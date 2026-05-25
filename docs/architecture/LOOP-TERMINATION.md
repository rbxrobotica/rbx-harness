# Loop Termination Policy

**Status**: Foundation
**Version**: 0.1

---

## Why loop termination is a first-class concern

An agent without a termination policy is an infinite loop waiting to happen.
Agents that produce new tasks, spawn sub-agents, or retry failed steps can
enter runaway execution patterns even with well-intentioned logic.

The Loop Termination Policy is not a safety net bolted on after the fact — it
is declared upfront in the Mission, evaluated at every step boundary, and
enforced by the runtime.

**The primary directive for every agent**: seek the end of the loop, not
the perpetuation of it. At every step boundary, the agent must evaluate
whether the mission is complete, blocked, or requires escalation — not
just whether there is more work to do.

---

## LoopTerminationPolicy fields

Declared in the Mission (see `spec/mission-execution.schema.json`).

| Field | Type | Description |
|---|---|---|
| `max_iterations` | int | Maximum total steps across the entire mission |
| `max_retries_per_step` | int | Maximum retry attempts for any single step |
| `max_context_budget_ratio` | float [0,1] | Max fraction of total budget that may be consumed |
| `max_cost_usd` | float or null | Optional cost ceiling in USD |
| `timeout_seconds` | int or null | Wall-clock timeout for the entire mission |
| `completion_criteria` | CompletionCriterion[] | Conditions that constitute successful completion |
| `abort_criteria` | AbortCriterion[] | Conditions that require immediate mission abort |
| `escalation_policy` | EscalationPolicy | What to do when limits are approached |
| `loop_detection_window` | int | Number of recent steps to check for repetition |

---

## Completion criteria

A mission is considered **successfully complete** when all declared
`completion_criteria` are satisfied. The mission transitions to `completed`
status when all criteria pass.

| Criterion type | Description |
|---|---|
| `objective_achieved` | All expected outputs declared in the mission exist |
| `expected_outputs_produced` | All declared `expected_output_type`s have output refs |
| `postconditions_satisfied` | All step postconditions passed |
| `independent_review_approved` | A Reviewer Agent (distinct from executing agent) approved |
| `minimum_tests_passed` | Test step produced results meeting the declared pass threshold |
| `no_blocking_pendencies` | No steps are in `blocked` status |
| `human_approval_received` | Final `approve` step completed with `approved` decision |
| `trace_emitted` | Mission completion event was successfully persisted |

A mission must not self-certify completion without satisfying all declared
criteria. If criteria require an independent reviewer, the agent cannot skip
that gate by declaring the mission done unilaterally.

---

## Abort criteria

An abort criterion triggers immediate mission termination with status `aborted`.
The mission cannot continue. Abort events are persisted and emitted as
`mission.aborted` events for Thalamus.

| Criterion type | Description |
|---|---|
| `max_iterations_exceeded` | Total steps exceeded `max_iterations` |
| `budget_exhausted` | Context budget consumed beyond `max_context_budget_ratio` |
| `cost_limit_reached` | Token cost exceeded `max_cost_usd` |
| `timeout_elapsed` | Mission exceeded `timeout_seconds` |
| `loop_detected` | Same output produced in `loop_detection_window` consecutive steps |
| `context_insufficient` | Context too depleted to make meaningful progress |
| `decision_conflict` | Conflicting decisions from independent agents cannot be reconciled |
| `forbidden_action_committed` | An agent committed a forbidden action (audit log irrecoverable) |
| `escalation_limit_exceeded` | Max escalation attempts exhausted without resolution |
| `operator_abort` | Operator issued an explicit abort command |

---

## Step-boundary self-evaluation

At every step boundary, the executing agent must evaluate:

```
1. Mission complete?
   → Is every completion_criterion satisfied?
   → If yes: emit mission.completed, transition to completed.

2. Mandatory output missing?
   → Is there an expected_output that is not yet produced?
   → If yes: identify which step produces it, check if it is blocked.

3. Blocked?
   → Are any unresolved blockers preventing progress?
   → If yes: emit human_approval.required or review.required as appropriate.
   → Do not continue generating new tasks to work around the block.

4. Escalation required?
   → Is this a risk/conflict/safety situation beyond the agent's authority?
   → If yes: emit escalation.required, do not proceed.

5. Summarize and close session?
   → Is the current session's working context too large for continuation?
   → If yes: open a summarization session, compress, then open a new session.

6. Open independent review?
   → Does any output require a Reviewer Agent before next step?
   → If yes: emit review.required, pause until review.completed.

7. Stop for safety?
   → Is there any indication that proceeding would violate a guard?
   → If yes: emit guard.blocked_action, halt, escalate.
```

This self-evaluation is not optional or advisory. It is a required postcondition
of every step in the execution graph. An agent that produces new tasks without
running this evaluation is violating the loop termination contract.

---

## Loop detection

The runtime tracks recent step outputs in a sliding window of size
`loop_detection_window` (default: 5). If the same effective output appears in
`loop_detection_window` consecutive steps, a `loop.termination.detected` event
is emitted and the mission transitions to `aborted`.

Repetition detection is based on a canonical hash of:
- The step's `prompt_type`
- The step's `input_context_refs` (sorted)
- The step's `expected_output_type`
- A semantic similarity check (future: embedding distance)

A hash collision without semantic repetition is not treated as a loop.

---

## Escalation before abort

Before aborting, the mission evaluates its `escalation_policy`. If escalation
is possible (a human or operator can unblock the mission), the runtime:

1. Emits `escalation.required` with full context.
2. Pauses mission execution.
3. Waits up to `escalation_policy.timeout_before_abort` seconds.
4. If response received and mission unblocked: resume from current step.
5. If timeout elapsed: abort.

This ensures that limit-approaching conditions give humans a chance to
intervene before unrecoverable state is reached.

---

## Default policy (recommended baseline)

```yaml
max_iterations: 50
max_retries_per_step: 3
max_context_budget_ratio: 0.9
max_cost_usd: null
timeout_seconds: 3600
loop_detection_window: 5
completion_criteria:
  - type: expected_outputs_produced
  - type: no_blocking_pendencies
abort_criteria:
  - type: max_iterations_exceeded
  - type: budget_exhausted
  - type: loop_detected
  - type: operator_abort
escalation_policy:
  escalate_to: human
  timeout_before_abort: 300
```

Complex missions (multi-day, multi-repository) should declare explicit
`completion_criteria` tailored to their objective rather than relying on
defaults.

---

## Relation to control loop

Loop termination evaluation happens at two levels:

**Harness control loop level** (per-cycle, `runtime/loop/spec.md`):
- Circuit breaker, pause, and resume govern individual execution cycles.
- Stage budgets and queue depth govern real-time throughput.

**Mission Execution Graph level** (per-step, this document):
- `LoopTerminationPolicy` governs the mission-level view.
- Step-boundary self-evaluation governs agent reasoning.

Both operate. A cycle may complete cleanly while the mission-level loop is
accumulating toward its `max_iterations` limit.

---

## Related documents

- `spec/mission-execution.schema.json` — LoopTerminationPolicy and mission fields
- `spec/orchestration-events.schema.json` — loop.termination.detected, mission.aborted
- `docs/architecture/MISSION-EXECUTION-GRAPH.md` — Step boundary and conditions
- `runtime/loop/spec.md` — Control loop circuit breaker and interruption
