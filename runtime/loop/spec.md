# Control Loop Specification

**Version**: 1.0  
**Status**: APPROVED  
**Derived from**: [Robson v3 Control Loop](https://github.com/rbxrobotica/robson/blob/main/docs/architecture/v3-control-loop.md)

---

## Overview

The Control Loop is the heartbeat of every rbx-harness agent. It is the single execution path through which ALL agent behavior flows.

```
Observe → Interpret → Decide → Act → Evaluate → Persist
```

The runtime is the EXCLUSIVE owner of the Control Loop. No external component may initiate, pause, interrupt, or restart a cycle outside the runtime.

Each cycle begins with a single trigger (an Observation), passes through six sequential stages, and ends with an immutable EventLog append.

---

## Stage specification

### Stage 1: Observe

**Deterministic**: Yes  
**Side effects**: None

**Input**: Raw external event  
**Output**: Typed `Observation`

The Observe stage parses raw input into a strongly-typed Observation. Parsing is deterministic. Malformed input is logged as `ObservationError` and the cycle is dropped — it does not proceed to Interpret.

Observation types are agent-specific. Examples:

```rust
// Generic observation types
pub enum Observation {
    ExternalSignal { signal_id: Ulid, payload: Value, source: String, timestamp: DateTime<Utc> },
    OperatorCommand { command: String, params: Value, issued_at: DateTime<Utc> },
    TimerFire { interval_id: String, fired_at: DateTime<Utc> },
    InternalEvent { event_type: String, data: Value },
}
```

---

### Stage 2: Interpret

**Deterministic**: Yes  
**Side effects**: None

**Input**: `Observation` + current `RuntimeState`  
**Output**: Typed `Interpretation`

The Interpret stage enriches the Observation with current state. Given the same Observation and RuntimeState, it always produces the same Interpretation. This stage may return `NoAction` if the observation is valid but irrelevant to the current state.

---

### Stage 3: Decide

**Deterministic**: Yes  
**Side effects**: None — **pure function**

**Input**: `Interpretation` + agent's declared limits  
**Output**: `EngineAction`

The Engine is a pure function. It makes a decision based solely on its inputs. It has no access to I/O, clocks, or randomness.

**Sandbox Gate** (sub-stage): Before the EngineAction is accepted, the sandbox evaluates it against the agent's manifest rules:
1. Does the manifest permit this action type?
2. Does this action violate any declared `cannot_do` rule?
3. Are resource budgets respected?

If denied: `EngineAction` is replaced with a `Blocked` variant. A `PermissionCheck { decision: denied }` event is queued for persistence.

---

### Stage 4: Act

**Deterministic**: **No**  
**Side effects**: Yes — external system calls

**Input**: Approved `EngineAction`  
**Output**: `ActionResult`

This is the only stage where external calls happen: LLM APIs, HTTP requests, database writes, exchange APIs, etc. Non-determinism from network latency, external state, and timing is contained here.

**Retry policy**: Configurable per action type. Default: 3 attempts with exponential backoff (100ms, 500ms, 2s). After exhausting retries: `ActionResult::Failed { retryable: false }`.

**Audit**: Full request and response are logged. External correlation IDs (e.g., exchange order IDs) are captured for reconciliation.

**Governance check**: Before executing, the runtime evaluates `governance.human_review_required` conditions. If a blocking condition is triggered, Act pauses and a `ReviewTriggered` event is persisted. Execution resumes only after `ReviewCompleted { decision: approved }`.

---

### Stage 5: Evaluate

**Deterministic**: Yes  
**Side effects**: None

**Input**: `ActionResult` + current `RuntimeState`  
**Output**: `(StateTransition, Vec<DomainEvent>)`

Given the same ActionResult and state, Evaluate always produces the same transition and events. This stage computes the new state and enumerates the domain events to be persisted.

---

### Stage 6: Persist

**Deterministic**: Yes  
**Side effects**: Appends to EventLog

**Input**: `Vec<DomainEvent>` from Evaluate  
**Output**: `Vec<EventEnvelope>` (events with assigned IDs, sequence numbers, timestamps)

The EventLog is append-only. Events are deduplicated by SHA256 hash of the canonical payload. Every cycle persists at minimum:

1. `harness.cycle_started`
2. `harness.permission_check` (if sandbox evaluated an action)
3. `governance.review_triggered` (if applicable)
4. Domain events from Evaluate (0..N)
5. `harness.cycle_completed`

**Failure handling**: If Persist fails (database unavailable), the runtime halts all execution, caches events in a bounded in-memory buffer (capacity: 100 events), and retries. If retries are exhausted: enter error state, alert operator.

---

## Cycle triggers

| Priority | Source | Queue behavior |
|---|---|---|
| Critical | Circuit breaker | Preempts current cycle at next safe point |
| High | Operator command | Front of queue |
| High | Risk / safety alert | Front of queue |
| Normal | External signals | FIFO |
| Normal | LLM responses | FIFO |
| Low | Timer / housekeeping | FIFO, droppable when queue > 800 |

**Queue**: Bounded channel, capacity 1000. Drops oldest Low priority items when depth > 800. Fill events and operator commands are never dropped. `QueueOverflow` event logged on any drop.

---

## Interruption protocol

### Pause

```
Operator issues PAUSE
→ Current cycle completes (all 6 stages)
→ After Persist: RuntimeState.paused = true
→ Observe stage skips all Non-critical observations
→ PauseActivated event persisted
```

### Resume

```
Operator issues RESUME
→ RuntimeState.paused = false
→ Normal queue processing resumes
→ ResumeActivated event persisted
```

### Circuit Breaker

```
Safety threshold breached
→ CircuitBreakerActivated event persisted
→ Current cycle completes Evaluate + Persist
→ Subsequent cycles: all state-mutating actions automatically denied by sandbox
→ Only operator CircuitBreakerReset restores normal operation
```

### Crash Recovery

```
Runtime restarts
→ Replay EventLog to reconstruct RuntimeState
→ Query external systems for actual state
→ Compare: EventLog state vs external state
→ If match: resume normal operation
→ If mismatch:
   - Adopt external state as truth
   - Persist ReconciliationEvent with discrepancy details
   - Alert operator
   - Resume with reconciled state
```

---

## Timing constraints

| Stage | Target latency | Metric |
|---|---|---|
| Full cycle (no external call) | < 50ms | `harness_cycle_duration_ms{type=internal}` |
| Full cycle (with external call) | < 500ms | `harness_cycle_duration_ms{type=external}` |
| Observe + Interpret + Decide | < 10ms combined | Per-stage histograms |
| Persist | < 5ms | `harness_persist_duration_ms` |
| Queue drain latency | < 10ms at p99 | `harness_queue_latency_ms` |

Latency violations are emitted as metrics and included in the `harness.cycle_completed` event. They do not abort the cycle.

---

## Observability

### Prometheus metrics

```
harness_cycles_total{agent_id, trigger_type}        # Counter
harness_cycle_duration_ms{agent_id, type}           # Histogram
harness_stage_duration_ms{agent_id, stage}          # Histogram
harness_queue_depth{agent_id}                       # Gauge
harness_permission_decisions_total{agent_id, decision}  # Counter
harness_circuit_breaker_state{agent_id}             # Gauge: 0=closed, 1=open
harness_eventlog_sequence{agent_id}                 # Gauge
harness_reviews_pending{agent_id}                   # Gauge
```

### Structured log (per cycle)

```json
{
  "cycle_id": "01HXYZ...",
  "agent_id": "robson-v3",
  "trigger_type": "external_signal",
  "outcome": "completed",
  "stages": {
    "observe_ms": 1,
    "interpret_ms": 2,
    "decide_ms": 1,
    "act_ms": 340,
    "evaluate_ms": 1,
    "persist_ms": 4
  },
  "events_produced": 3,
  "permission_decision": "allowed",
  "queue_depth_at_start": 4
}
```

### OpenTelemetry traces

Each cycle is a trace span. Stage spans are children. `trace_id` correlates with the Thalamus protocol `trace_id` from the originating message.

---

## Origin note

This specification is a generalization of Robson v3's control loop (`docs/architecture/v3-control-loop.md`). The Robson loop was domain-specific (trading: MarketTick, DetectorSignal, OrderFill). This specification replaces domain types with generic types and adds the governance layer as a first-class primitive.

Robson v3's loop proved that:
- Deterministic stages 1–3 and 5 enable reliable replay and testing
- Isolating non-determinism to stage 4 makes the system debuggable
- The EventLog-as-state-source enables crash recovery without data loss
- Priority queues with bounded capacity prevent cascading overload
