# Mission Execution Graph

**Status**: Foundation
**Version**: 0.1

---

## Overview

The Mission Execution Graph (MEG) is the model for how agent work is structured,
ordered, and governed. It treats prompt execution as a **sequenced, governed
graph** rather than an open-ended conversation.

Every unit of governed agent work is a **Mission**. Every step within a mission
is a **PromptExecutionStep**. Steps have declared dependencies, preconditions,
postconditions, and stop conditions. The mission has a declared
**LoopTerminationPolicy** that prevents runaway execution.

See `spec/mission-execution.schema.json` for the machine-readable schema.

---

## Mission

A Mission is the top-level unit of governed agent work. It is created by a
Mission Orchestrator and may contain one or more sessions, each of a specific
session type.

### Mission fields

| Field | Type | Description |
|---|---|---|
| `mission_id` | ULID | Unique mission identifier |
| `name` | string | Human-readable mission name |
| `mission_type` | enum | classification (see below) |
| `objective` | string | What the mission is trying to accomplish |
| `scope` | object | Repositories, domains, systems in scope |
| `constraints` | object | Hard constraints (budget, time, forbidden actions) |
| `owner_agent_id` | string | Mission Orchestrator agent ID |
| `context_budget_policy_id` | string | Which budget policy applies |
| `loop_termination_policy` | object | Termination and abort criteria |
| `created_at` | RFC3339 | — |
| `started_at` | RFC3339 or null | — |
| `completed_at` | RFC3339 or null | — |
| `status` | enum | pending / running / paused / blocked / completed / aborted / escalated |
| `parent_mission_id` | ULID or null | For sub-missions spawned by an orchestrator |
| `steps` | PromptExecutionStep[] | Ordered step graph |
| `completion_evidence` | object | What was produced; links to outputs |

### Mission types

| Type | Description |
|---|---|
| `implement` | Produce working code or configuration |
| `research` | Gather and synthesize information |
| `review` | Evaluate existing work for correctness or compliance |
| `architecture` | Produce design proposals or ADRs |
| `test` | Design and execute tests |
| `summarize` | Compress prior work into durable summaries |
| `governance` | Validate policies, check invariants, audit trail |
| `deployment_readiness` | Evaluate readiness for production |
| `incident_analysis` | Post-incident review and remediation |
| `commercial` | Sales, pricing, or commercial messaging |
| `support` | User-facing response drafting |

---

## Session Types

A session is an isolated execution context for one session type. Sessions within
a mission do not share raw context — only what is explicitly passed through
declared outputs.

| Session Type | Carries | Must not carry |
|---|---|---|
| `planning` | objective, constraints, ADR refs | verbose logs, test output |
| `research` | selected sources, search results | implementation diffs |
| `architecture` | ADRs, schemas, design context | test logs, raw error output |
| `coding` | relevant source files, current plan | full research history |
| `review` | diff, test results, review criteria | implementation noise |
| `testing` | hypothesis, commands, relevant code, results | planning history |
| `summarization` | full session history from prior session | — |
| `writing` | outline, references, style guide | build logs |
| `deployment_readiness` | checklist, manifests, test results | development history |
| `incident_analysis` | timeline, logs, alerts | unrelated features |
| `commercial` | product context, audience profile | code diffs |
| `support` | incident context, user history | internal logs |
| `governance` | decisions, contracts, audit trail | raw dev logs |

A summarization session takes the history of a prior session as input and
produces a compressed summary that can be loaded by future sessions. This is
the mechanism for context continuity across sessions without context blowup.

---

## PromptExecutionStep

A PromptExecutionStep is a single declared step in the mission's execution
graph. It is not an ad-hoc prompt — it is a governed unit with declared
inputs, expected outputs, conditions, and status.

### Step fields

| Field | Type | Description |
|---|---|---|
| `prompt_id` | ULID | Unique step identifier |
| `mission_id` | ULID | Parent mission |
| `session_id` | ULID | Session this step runs in |
| `agent_id` | string | Agent responsible for this step |
| `prompt_type` | enum | Type of work (see below) |
| `input_context_refs` | string[] | IDs of context items, files, or step outputs needed |
| `expected_output_type` | string | What this step must produce |
| `dependencies` | ULID[] | Step IDs that must complete before this step starts |
| `execution_order` | int | Ordinal within dependency-resolved set |
| `preconditions` | Condition[] | Must be true before the step begins |
| `postconditions` | Condition[] | Must be true after the step completes |
| `stop_conditions` | Condition[] | If true mid-execution, stop immediately |
| `retry_policy` | RetryPolicy | Retry behavior on failure |
| `escalation_policy` | EscalationPolicy | What to do if blocked or failed |
| `status` | enum | pending / running / completed / failed / blocked / skipped / cancelled |
| `trace_id` | ULID | Links to Thalamus trace for this step's AI call |
| `started_at` | RFC3339 or null | — |
| `completed_at` | RFC3339 or null | — |
| `output_ref` | string or null | Where the step output was persisted |
| `failure_reason` | string or null | If status is failed or blocked |

### Prompt types

| Type | Description |
|---|---|
| `plan` | Produce an execution plan for a mission or sub-mission |
| `research` | Gather information about a topic |
| `inspect` | Read and analyze existing code or documents |
| `implement` | Produce a code change or configuration |
| `test` | Execute tests and report results |
| `review` | Evaluate a diff, proposal, or output |
| `summarize` | Compress a session's history into durable form |
| `decide` | Evaluate options and select one with rationale |
| `approve` | Human or governance agent approval gate |
| `deploy_readiness` | Check all gates before deploy |
| `rollback_analysis` | Assess rollback risk and procedure |
| `commercial_draft` | Draft a commercial message or proposal |
| `support_draft` | Draft a user-facing support response |

---

## Execution Conditions

### Preconditions

A step may declare preconditions that must be satisfied before it begins.
If any precondition fails, the step moves to `blocked` status.

| Condition type | Description |
|---|---|
| `repo_available` | Required repository is accessible |
| `branch_correct` | Current branch is the expected one |
| `prior_tests_passed` | Tests from a prior step passed |
| `adr_exists` | Required architectural decision exists |
| `min_context_present` | Minimum required context items are loaded |
| `budget_sufficient` | Estimated tokens for this step fit within remaining budget |
| `tool_permitted` | Required tool is in the agent's `allowed_tools` |
| `human_approval_received` | A prior `approve` step completed with decision: approved |
| `dependency_completed` | All declared dependencies are in `completed` status |
| `agent_authorized` | Agent's domain contract permits this step type |

### Postconditions

Postconditions are evaluated after a step completes. If a postcondition is not
met, the step is re-evaluated (within retry limits) or escalated.

| Condition type | Description |
|---|---|
| `output_produced` | Step output exists and is non-empty |
| `tests_executed` | Test step ran at least one test |
| `summary_generated` | Summarization step produced a summary |
| `trace_emitted` | Orchestration event was emitted |
| `diff_in_scope` | Diff does not touch files outside declared scope |
| `no_forbidden_action` | No `forbidden_actions` were executed |
| `next_step_identified` | Next step in the graph is known and unblocked |
| `mission_status_updated` | Mission state reflects step outcome |

### Stop Conditions

Stop conditions are evaluated continuously during step execution. If triggered,
the step halts immediately and emits a `loop.termination.detected` event.

| Condition type | Description |
|---|---|
| `budget_exceeded` | Step would exceed context budget ceiling |
| `repetitive_output` | Step is producing the same output as a prior step |
| `forbidden_action_attempted` | Agent attempted a prohibited action |
| `out_of_scope_access` | Agent accessed a repository or tool not in contract |
| `review_required_but_missing` | Output requires review but no reviewer is available |
| `escalation_threshold_reached` | Max escalation attempts exceeded |
| `cost_limit_reached` | Estimated token cost exceeded session limit |

---

## Execution modes

Steps within a mission may execute in these modes:

| Mode | Description |
|---|---|
| `linear` | Steps execute in `execution_order` sequence |
| `dependency_graph` | Steps execute when all declared `dependencies` are completed |
| `parallel_safe` | Steps with no shared scope may execute concurrently |
| `human_gated` | Execution pauses at `approve` step type pending human decision |
| `reviewer_gated` | Execution pauses pending a Reviewer Agent's decision |

Parallel execution requires that the steps do not share `allowed_repositories`,
`allowed_tools`, or `allowed_memory_scopes`. The runtime enforces this via the
sandbox gate.

---

## Step lifecycle

```
pending
  │
  ├── preconditions checked
  │     ├── pass → running
  │     └── fail → blocked (await precondition resolution)
  │
running
  │
  ├── stop_condition triggered → blocked or failed
  │
  ├── step completes
  │     ├── postconditions pass → completed
  │     └── postconditions fail → retry (within retry_policy)
  │           └── retries exhausted → failed → escalation_policy evaluated
  │
blocked
  │
  ├── orchestrator or operator resolves block → pending (re-check preconditions)
  └── escalation_policy: escalate_to_human → emit human_approval.required event

cancelled
  └── explicit cancellation by orchestrator or operator
```

---

## Retry and escalation policies

### RetryPolicy

| Field | Description |
|---|---|
| `max_retries` | Maximum retry attempts for this step |
| `backoff_strategy` | fixed / exponential / none |
| `initial_delay_ms` | Starting delay before first retry |
| `max_delay_ms` | Ceiling on backoff delay |
| `retryable_on` | Which failure reasons trigger a retry |

### EscalationPolicy

| Field | Description |
|---|---|
| `escalate_to` | human / reviewer_agent / mission_orchestrator |
| `escalation_reason_template` | Message template for the escalation |
| `timeout_before_abort` | How long to wait before aborting if no response |
| `abort_action` | cancel_step / abort_mission / pause_mission |

---

## Related documents

- `spec/mission-execution.schema.json` — Machine-readable schema
- `spec/session.schema.json` — Session type constraints
- `docs/architecture/LOOP-TERMINATION.md` — Mission-level termination policies
- `docs/architecture/CONTEXT-MANAGEMENT.md` — Context selection per step
- `docs/architecture/AGENT-DOMAIN-CONTRACT.md` — Domain Contract evaluation
