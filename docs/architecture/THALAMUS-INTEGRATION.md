# Thalamus Integration

**Status**: Foundation
**Version**: 0.1

---

## Boundary recap

Thalamus is the **semantic control layer for AI traffic** (ADR-0001,
`thalamus-core`). It owns pre-call decisions and post-call validation for every
AI-mediated call. The AOP owns mission governance, context management, memory,
and orchestration traceability.

**The AOP does not replicate Thalamus**. It emits structured orchestration
events that Thalamus (and Strategos, rbx-console) can consume for observation,
audit, and evaluation.

```
AOP runtime
  │ emits orchestration events
  ▼
Thalamus
  │ consumes for: tracing, audit, policy enforcement, evaluation
  ▼
Strategos / rbx-console / rbx-governance
  │ consumes for: mission tracking, operational visibility, compliance
```

---

## Event envelope extension

All orchestration events use the base `EventEnvelope` from
`spec/governance.schema.json`, extended with AOP-specific fields:

```json
{
  "event_id": "<ULID>",
  "event_type": "<namespace.event_name>",
  "agent": { "agent_id": "...", "agent_version": "..." },
  "cycle_id": "<ULID>",
  "timestamp": "<RFC3339>",
  "sha256": "<hash>",
  "payload": {
    "trace_id": "<ULID>",
    "mission_id": "<ULID or null>",
    "session_id": "<ULID or null>",
    "prompt_id": "<ULID or null>",
    "parent_step_id": "<ULID or null>",
    "severity": "info | warn | error | critical",
    "reason": "<human-readable summary>",
    "metadata": {}
  }
}
```

`metadata` must never contain secrets, credentials, or PII. The runtime
enforces this via the output guardrail (see §Output guardrails below).

---

## Orchestration event catalog

### Mission events

| Event type | Emitted when |
|---|---|
| `mission.created` | Mission record is created (before execution starts) |
| `mission.started` | Mission transitions from `pending` to `running` |
| `mission.paused` | Mission execution paused (operator or governance gate) |
| `mission.resumed` | Mission execution resumed after pause |
| `mission.completed` | All completion criteria satisfied |
| `mission.aborted` | An abort criterion triggered |
| `mission.escalated` | Escalation required; awaiting human or operator |

### Session events

| Event type | Emitted when |
|---|---|
| `session.created` | A new session is instantiated within a mission |
| `session.started` | First step in the session begins |
| `session.closed` | Session ends normally (steps complete) |
| `session.expired` | Session TTL elapsed before completion |

### Step events

| Event type | Emitted when |
|---|---|
| `mission.step.started` | A PromptExecutionStep transitions to `running` |
| `mission.step.completed` | Step postconditions satisfied |
| `mission.step.failed` | Step failed after retries exhausted |
| `mission.step.blocked` | Step blocked (precondition failed or guard triggered) |
| `mission.step.skipped` | Step skipped due to a dependency branch decision |
| `mission.step.retrying` | Step retrying after failure |

### Context budget events

| Event type | Emitted when |
|---|---|
| `context.budget.estimated` | Context budget estimated for a step before execution |
| `context.budget.compression_applied` | Compression was applied to fit within ceiling |
| `context.budget.exceeded` | Step could not fit within ceiling even after compression |

### Prompt execution events

| Event type | Emitted when |
|---|---|
| `prompt.execution.started` | AI model call initiated for a step |
| `prompt.execution.completed` | AI model call returned a response |
| `prompt.execution.timed_out` | AI model call exceeded timeout |

### Tool events

| Event type | Emitted when |
|---|---|
| `tool.call.requested` | Agent requested a tool call |
| `tool.call.completed` | Tool call returned a result |
| `tool.call.denied` | Tool call blocked by Domain Contract or Sandbox Gate |
| `tool.call.failed` | Tool call returned an error |

### Memory events

| Event type | Emitted when |
|---|---|
| `memory.read` | Agent retrieved a memory item |
| `memory.write` | Agent persisted a new memory item |
| `memory.scope.violation` | Agent attempted to access memory outside `allowed_memory_scopes` |

### Guard events

| Event type | Emitted when |
|---|---|
| `guard.blocked_action` | A non-overridable guard prevented an action |
| `guard.secret_in_context` | A potential secret was detected in assembled context |
| `guard.scope_exceeded` | Agent accessed a repo, tool, or scope outside contract |
| `guard.self_review_attempted` | Agent attempted to review its own output |
| `guard.permission_escalation_attempted` | Sub-agent attempted permissions exceeding orchestrator |

### Human approval and review events

| Event type | Emitted when |
|---|---|
| `human_approval.required` | A step requires human gate before proceeding |
| `human_approval.received` | Human approved (decision: approved / rejected / modified) |
| `review.required` | An output requires Reviewer Agent evaluation |
| `review.assigned` | Review assigned to a Reviewer Agent |
| `review.completed` | Reviewer Agent returned decision |

### Loop termination and escalation events

| Event type | Emitted when |
|---|---|
| `loop.termination.detected` | Repetitive output detected in detection window |
| `loop.iteration.limit_approached` | Iterations at 80% of `max_iterations` |
| `escalation.required` | Escalation triggered; execution paused |
| `escalation.resolved` | Escalation resolved by human or operator |
| `escalation.timed_out` | Escalation timeout elapsed; mission aborting |

---

## What Thalamus observes vs. what it owns

| Concern | Owner | AOP role |
|---|---|---|
| Pre-call: model/tool routing decision | Thalamus | Emits `prompt.execution.started` with `trace_id` |
| Pre-call: context authorization | Thalamus | Enforces budget ceiling; emits `context.budget.*` |
| Pre-call: secret redaction | Thalamus (call level) | AOP emits `guard.secret_in_context` before call |
| Post-call: output schema validation | Thalamus | AOP emits scope/guard events if output violates contract |
| Post-call: hallucination detection | Thalamus | Not duplicated by AOP |
| Post-call: audit event | Thalamus | AOP emits step-level audit via `mission.step.*` |
| Mission lifecycle | AOP | Thalamus observes via `mission.*` events |
| Memory access patterns | AOP | Thalamus observes via `memory.*` events |
| Agent permission enforcement | AOP (Sandbox Gate) | Thalamus observes via `guard.*` and `harness.permission_check` |
| Human review gates | AOP | Thalamus observes via `human_approval.*` and `review.*` |

---

## Strategos integration

Strategos consumes orchestration events to populate:

- **Situation Room**: active missions, current step, last event
- **Mission registry**: mission lifecycle history, completion evidence
- **Escalation tracker**: open escalations, owner, SLA
- **Review queue**: pending reviews, reviewer assignments

Strategos does not initiate missions through the AOP directly. A human operator
or a Strategos planning agent uses the Mission Orchestrator to create missions
via the harness runtime API (future: `rbx-harness` REST/gRPC endpoint).

---

## rbx-governance integration

`rbx-governance` consumes:

- `guard.blocked_action` — evidence of policy enforcement
- `guard.permission_escalation_attempted` — policy violation attempts
- `human_approval.*` — audit trail of governance decisions
- `mission.completed` + `completion_evidence` — mission deliverables for ADR/policy registry

Domain Contracts reference `rbx-governance` policy IDs. At mission creation,
the runtime validates that referenced policies exist and are in `accepted` status.

---

## rbx-console integration

`rbx-console` is the operator UI. It subscribes to the orchestration event
stream and surfaces:

- Mission status and step graph (with current step highlighted)
- Context budget utilization per session
- Pending reviews and human approvals (with action buttons)
- Guard events and blocked actions
- Loop termination warnings
- Cost and token usage per mission

The AOP does not implement the console. It produces the event stream that makes
the console meaningful.

---

## Output guardrails (AOP responsibility before Thalamus post-call)

Before an output is submitted for Thalamus post-call validation, the AOP
runtime applies these output guardrails:

1. **No secrets in output**: Scan for credential patterns (keys, tokens,
   passwords) in the assembled output payload. If detected: emit
   `guard.secret_in_context`, block the submission, escalate.

2. **Scope adherence**: If the output contains a diff, validate that all
   modified file paths are within `allowed_repositories` and `domain` scope.
   Violations emit `guard.scope_exceeded`.

3. **No unauthorized approval claims**: Output must not assert that a human
   approved, a review passed, or a gate was cleared unless the corresponding
   `human_approval.received` or `review.completed` event exists in the
   EventLog. Violations emit `guard.blocked_action { reason: unauthorized_approval_claim }`.

4. **Reviewer independence**: If the output requires review (per
   `review_requirements`), the assigned reviewer must have a different
   `agent_id` than the agent that produced the output. The runtime enforces
   this at `review.assigned` time.

These guardrails run before the output is acted on and before Thalamus
post-call validation. They are the AOP's last line of defense at the output
boundary.

---

## Trace ID propagation

Every `prompt.execution.started` event carries a `trace_id` that links to
the Thalamus trace for that AI call. This trace ID must be included in:

- The corresponding `prompt.execution.completed` event
- Any `guard.*` or `tool.*` events produced during the step
- The `mission.step.completed` or `mission.step.failed` event

This creates a navigable chain from mission-level events down to individual
AI call traces in Thalamus, enabling full end-to-end observability.

---

## Related documents

- `spec/orchestration-events.schema.json` — Machine-readable event schemas
- `spec/governance.schema.json` — Base event envelope (harness.* and governance.*)
- `spec/protocol.md` — Thalamus protocol (message envelope, trace_id)
- `thalamus-core/docs/adr/ADR-0001-thalamus-as-semantic-control-layer.md`
- `docs/architecture/AGENT-ORCHESTRATION-PLANE.md` — AOP overview and boundaries
