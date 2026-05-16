# ARCHITECTURE — rbx-harness

---

## Overview

rbx-harness is composed of three independent layers that can be adopted separately:

```
┌──────────────┐
│     CLI      │  rbx-harness new agent | validate manifest | register
└──────┬───────┘
       │ uses
┌──────▼───────┐
│     Spec     │  manifest.schema.json | protocol.md | governance.schema.json
└──────┬───────┘
       │ implemented by
┌──────▼───────┐     ┌────────────────────────────┐
│   Runtime    │     │           SDKs             │
│   (Rust)     │     │  Go SDK  |  TypeScript SDK  │
└──────────────┘     └────────────────────────────┘
```

The **spec** is the canonical source of truth. The runtime and SDKs are implementations of the spec.

---

## Layer 1: Spec

The spec layer defines three contracts:

### 1.1 Agent Manifest (`spec/manifest.schema.json`)

A YAML or JSON document that every RBX agent must provide. It declares:

- **Identity**: id, name, version, status, role
- **Capabilities**: inputs, processing modes, outputs
- **Limitations**: cannot_do, should_defer, known_limitations
- **Interface**: protocol version, request/response schemas, timeout, auth
- **Governance**: human_review_required conditions, audit_level, approval gates
- **Resources**: compute budget, token budget, storage
- **Operational**: owner, monitoring thresholds, deployment strategy

The manifest is validated before agent activation. An agent with an invalid manifest cannot be started.

### 1.2 Thalamus Protocol (`spec/protocol.md`)

The wire protocol between agents and the Thalamus mediation layer. Defines:

- Message envelope format
- Request/response lifecycle
- Signal routing semantics
- Error codes and retry semantics

### 1.3 Governance Schema (`spec/governance.schema.json`)

Defines the structure of governance events:

- Human review trigger conditions
- Approval workflow states
- Audit log event envelope

---

## Layer 2: Runtime (Rust)

The runtime implements the **control loop** — the single execution path through which all agent behavior flows.

### Control loop stages

```
Observe → Interpret → Decide → Act → Evaluate → Persist
```

| Stage | Deterministic | Owner | Side effects |
|---|---|---|---|
| Observe | Yes | Runtime | None — parses raw input into typed Observation |
| Interpret | Yes | Runtime | None — enriches with current state |
| Decide | Yes | Engine (pure fn) | None — returns EngineAction |
| Act | **No** | Executor | Calls external systems |
| Evaluate | Yes | Runtime | None — computes state transition |
| Persist | Yes | EventLog | Appends to immutable log |

The Act stage is the only place where non-determinism is allowed. All other stages must be pure and testable without I/O.

### Runtime components

```
rbx-harness-runtime (Rust crate)
├── loop/         ← Orchestrates the 6-stage pipeline
├── sandbox/      ← Permission enforcement (what actions are allowed)
├── context/      ← Context budget management (token window, compression)
├── audit/        ← Immutable EventLog (append-only, SHA256 dedup)
└── governance/   ← Human review trigger evaluation
```

### Crash recovery

On restart, the runtime replays the EventLog to reconstruct RuntimeState, then reconciles with any external system state. Discrepancies are logged as ReconciliationEvents and the operator is alerted.

---

## Layer 3: SDKs

SDKs implement the Thalamus protocol for a given language, plus manifest validation utilities.

### Go SDK (`sdk/go/`)

For use in:
- Strategos Core (strategic orchestration engine)
- Thalamus (mediation layer implementation)
- Any Go-based agent or service in the RBX ecosystem

### TypeScript SDK (`sdk/typescript/`)

For use in:
- Frontend agents (browser or edge)
- rbx-catalog-console (agent registry UI)
- Éden IDP flows

---

## Layer 4: CLI

The `rbx-harness` CLI provides developer tooling:

```
rbx-harness new agent <name>       # Scaffold a new agent directory with manifest template
rbx-harness validate manifest      # Validate manifest.yaml against the spec schema
rbx-harness validate governance    # Check governance triggers for completeness
rbx-harness register <manifest>    # Register an agent in the RBX catalog (Éden integration)
rbx-harness diff <v1> <v2>         # Show breaking changes between two manifest versions
```

The CLI is the primary integration point with **Éden** (Internal Development Platform).

---

## Relation to Thalamus (the AI control plane)

Thalamus is the **semantic control layer for AI traffic** (canonical
definition: `thalamus-core`,
`docs/adr/ADR-0001-thalamus-as-semantic-control-layer.md`). It applies policy,
context authorization, validation, audit, and evaluation before and after
AI-mediated calls. It decides and validates; it does not transport bytes. The
transport (proxy, routing, rate limits) is the replaceable data plane below
Thalamus (Agentgateway/LiteLLM/etc.). The Thalamus protocol in
`spec/protocol.md` is the agent-facing view of that control plane.

The harness runtime and Thalamus are complementary:

```
Inbound message (via Thalamus protocol)
      │
      ▼
  Thalamus                 ← pre-call: identity, policy, context auth, model/
      │                       tool selection, budget, routing decision,
      │                       trace_id/audit_id; delegates transport to the
      │                       data plane. post-call: schema, risk,
      │                       hallucination, citations, audit, evaluation
      ▼
rbx-harness runtime        ← governs execution, enforces permissions, audits
      │
      ▼
  Agent Logic              ← domain-specific (Robson engine, Strategos planner, etc.)
```

Thalamus owns *whether a call is allowed, with which context, and whether the
result is acceptable*. The harness runtime owns *how the agent executes*.

---

## Versioning strategy

| Layer | Versioning |
|---|---|
| Spec schemas | Semantic versions in `$id` URL |
| Thalamus protocol | `MAJOR.MINOR` in manifest `interface.protocol_version` |
| Runtime crate | Semantic versioning (Cargo) |
| SDKs | Semantic versioning (go.mod, package.json) |
| CLI | Semantic versioning, Git tags |

Breaking changes in the spec trigger a major version bump and require migration guides.

---

## Case study: Robson v3

Robson v3's control loop (`Observe → Interpret → Decide → Act → Evaluate → Persist`) is the origin of the runtime specification. Key insights extracted from Robson that generalize to the harness:

1. **Determinism by stage**: making Observe/Interpret/Decide/Evaluate pure functions enables replay and testability
2. **Bounded queues with priority**: critical signals preempt normal signals without blocking the loop
3. **EventLog as state source of truth**: state is reconstructed from events, never assumed
4. **Risk Gate as sub-stage of Decide**: permission enforcement is inlined in the pipeline, not bolted on
5. **Timing budgets per stage**: each stage has a latency target; violations are metrics, not exceptions

See `runtime/loop/spec.md` for the full specification.
