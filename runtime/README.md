# rbx-harness runtime

**Language**: Rust  
**Status**: Not yet implemented — specification phase

The runtime is the execution engine of the RBX harness. It implements the control loop, sandboxing, governance evaluation, context budgeting, and the immutable EventLog.

---

## Status

The runtime specification is documented in `loop/spec.md`. This specification is derived from [Robson v3](https://github.com/rbxrobotica/robson)'s `docs/architecture/v3-control-loop.md` — the most battle-tested governed execution loop in the RBX portfolio.

Implementation will begin once the spec layer is stabilized.

---

## Planned crate structure

```
rbx-harness-runtime/         ← Rust crate (library + optional binary)
├── src/
│   ├── lib.rs
│   ├── loop_/               ← Control loop orchestration
│   │   ├── mod.rs
│   │   ├── observe.rs       ← Stage 1: parse raw input into typed Observation
│   │   ├── interpret.rs     ← Stage 2: enrich with current RuntimeState
│   │   ├── decide.rs        ← Stage 3: pure function → EngineAction
│   │   ├── act.rs           ← Stage 4: execute against external systems
│   │   ├── evaluate.rs      ← Stage 5: compute state transition
│   │   └── persist.rs       ← Stage 6: append to EventLog
│   ├── sandbox/             ← Permission enforcement
│   │   ├── mod.rs
│   │   └── evaluator.rs     ← Evaluates EngineAction against manifest rules
│   ├── governance/          ← Human review trigger evaluation
│   │   ├── mod.rs
│   │   └── trigger.rs       ← Evaluates governance conditions from manifest
│   ├── context/             ← Context budget management
│   │   ├── mod.rs
│   │   └── budget.rs        ← Token window and compression logic
│   ├── audit/               ← Immutable EventLog
│   │   ├── mod.rs
│   │   └── eventlog.rs      ← Append-only, SHA256 dedup
│   └── manifest/            ← Manifest loading and validation
│       ├── mod.rs
│       └── loader.rs
└── Cargo.toml
```

---

## Key design decisions

**1. Each stage is a typed transformation**

```rust
Observation  ->  Interpret  ->  Interpretation
Interpretation -> Decide   ->  EngineAction
EngineAction   -> Act      ->  ActionResult
ActionResult   -> Evaluate ->  (StateTransition, Vec<DomainEvent>)
Vec<DomainEvent> -> Persist -> Vec<EventEnvelope>
```

Types are the contract between stages. A stage cannot produce an invalid output for the next stage — the compiler enforces this.

**2. Stages 1–3 and 5 are pure functions**

Observe, Interpret, Decide, and Evaluate have zero I/O. They take inputs, return outputs, no side effects. This means:
- They are deterministically testable with no mocks
- They can be replayed from EventLog history
- Bugs in these stages are reproducible

**3. Act is the isolation zone for non-determinism**

All external calls (LLM APIs, exchange APIs, databases, HTTP) happen exclusively in Act. The rest of the loop is isolated from I/O.

**4. The sandbox evaluates before Act**

```
Decide → [Sandbox Gate] → Act
```

If the sandbox denies the action, a `PermissionCheck { decision: denied }` event is persisted and the cycle continues to Evaluate with a `Blocked` ActionResult. The agent cannot bypass this.

**5. EventLog is the source of truth**

RuntimeState is never stored directly. On startup, the runtime replays the EventLog to reconstruct state. This enables:
- Crash recovery without data loss
- Point-in-time state reconstruction for debugging
- Audit trails that are structurally impossible to modify

---

## Control loop specification

See `loop/spec.md` for the complete specification, including:
- Stage-by-stage input/output types
- Determinism guarantees
- Priority queue behavior
- Circuit breaker protocol
- Crash recovery procedure
- Timing constraints and observability targets
