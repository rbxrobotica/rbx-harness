# RBX Agent Orchestration Plane

**Status**: Foundation
**Version**: 0.1
**Repository**: rbx-harness

---

## What is the Agent Orchestration Plane?

The RBX Agent Orchestration Plane (AOP) is the governance layer that enables
AI agents to operate with **controlled autonomy** across the RBX ecosystem.

| Problem | What the AOP provides |
|---|---|
| Agents have no declared domain | Agent Domain Contract: scope, authority, forbidden actions |
| Context window fills uncontrolled | Context Budget Policy with an explicit 70% ceiling |
| Agent memory is opaque | Memory Taxonomy with source, scope, TTL, and confidence |
| No defined unit of agent work | Mission and Session as first-class governed primitives |
| Prompts execute in arbitrary order | Mission Execution Graph with dependencies and conditions |
| Agents loop indefinitely | Loop Termination Policy with explicit completion and abort criteria |
| No orchestration traceability | Structured events emitted to Thalamus for audit and observation |

The AOP lives in **rbx-harness**. The spec layer defines the contracts. The
runtime enforces them.

---

## Architectural position

```
┌──────────────────────────────────────────────────────────────────┐
│              rbx-console / Strategos Situation Room              │
│        (mission viewer, session list, cost, blocks, escalations) │
└───────────────────────────┬──────────────────────────────────────┘
                            │ orchestration events
┌───────────────────────────▼──────────────────────────────────────┐
│                   Thalamus (semantic control layer)              │
│  pre-call: identity, policy, model selection, context auth,      │
│            budget, redaction, routing, trace_id/audit_id         │
│  post-call: schema, risk, hallucination, audit, evaluation       │
└───────────────────────────┬──────────────────────────────────────┘
                            │ governed AI calls
┌───────────────────────────▼──────────────────────────────────────┐
│            Agent Orchestration Plane (rbx-harness)               │
│                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │ Agent Domain    │  │ Context Budget  │  │ Memory Taxonomy │  │
│  │ Contract        │  │ Management      │  │ (layered scopes)│  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │ Mission         │  │ Session         │  │ Loop            │  │
│  │ Execution Graph │  │ Isolation       │  │ Termination     │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
│                                                                  │
│  Control Loop: Observe → Interpret → Decide → Act →             │
│               Evaluate → Persist                                │
└───────────────────────────┬──────────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────────┐
│                      Agent Domain Logic                          │
│      (Robson engine, Strategos planner, Éden, domain agents)     │
└──────────────────────────────────────────────────────────────────┘
```

---

## Agent topology

An agent may occupy multiple roles across different missions. Role in a given
mission is declared in the Domain Contract (`authority_level`).

```
RBX Agent Orchestration Plane
└── Mission Orchestrator        — coordinates a mission, owns the execution graph
    ├── Domain Agent            — specialized by domain (code, architecture, research…)
    │   └── Task Agent          — executor of a single small step
    ├── Reviewer Agent          — independent review (never reviews own output)
    └── Governance Agent        — policy validation, invariant check, audit
```

| Role | Responsibility | Can orchestrate |
|---|---|---|
| Mission Orchestrator | Owns mission lifecycle and execution graph | Yes |
| Domain Agent | Specialized execution within a declared domain | Only within domain |
| Task Agent | Single-step executor | No |
| Reviewer Agent | Independent review, never self-approves | No |
| Governance Agent | Policy validation and audit | No |

A single agent binary may carry multiple domain contracts for different mission
roles. Which contract applies is determined at mission creation time.

---

## Core primitives

| Primitive | Schema | Purpose |
|---|---|---|
| Mission | `spec/mission-execution.schema.json` | Unit of governed work |
| Session | `spec/session.schema.json` | Isolated execution context by session type |
| PromptExecutionStep | `spec/mission-execution.schema.json` | Single step in the execution graph |
| Agent Domain Contract | `spec/domain-contract.schema.json` | Domain scope, authority, runtime permissions |
| Context Budget Policy | `spec/context-budget.schema.json` | Token allocation policy for a session |
| Memory Item | `spec/memory.schema.json` | Typed memory with scope, TTL, confidence |
| Orchestration Events | `spec/orchestration-events.schema.json` | Events emitted to Thalamus for tracing |

---

## Boundary with Thalamus

Thalamus owns **whether a call is allowed, with which context, and whether the
result is acceptable** (ADR-0001 in `thalamus-core`).

The AOP owns **how the agent is governed end-to-end**: mission structure,
context budget, memory taxonomy, loop termination, domain contract enforcement,
and orchestration traceability.

| Concern | Owner |
|---|---|
| Pre-call: model selection, context auth, routing decision | Thalamus |
| Post-call: response validation, hallucination detection | Thalamus |
| Agent domain permissions (can_push, allowed_tools, authority_level) | AOP |
| Context budget within a session (70% policy) | AOP |
| Memory taxonomy and TTL | AOP |
| Mission lifecycle and step ordering | AOP |
| Loop termination criteria | AOP |
| Orchestration events emitted for Thalamus to observe | AOP emits, Thalamus consumes |

The AOP does **not** replicate Thalamus's pre/post call validation. It emits
structured events that Thalamus can observe, trace, and evaluate.

---

## Boundary with Strategos

Strategos owns **strategic memory**: decision history, rationale, situation
room, and operational oversight. The AOP emits mission lifecycle events that
Strategos can consume for tracking, but the AOP does not replace Strategos's
planning and coordination functions.

---

## Boundary with Éden

Éden is the Internal Development Platform for agent scaffolding and lifecycle
management. The `rbx-harness` CLI is the integration point:
`rbx-harness new agent` and `rbx-harness register` feed agent definitions
into Éden's catalog.

---

## Boundary with rbx-governance

`rbx-governance` owns ADRs, policies, and the decision registry. Domain
Contracts reference governance policies by ID. The AOP does not replicate
policies — it consumes them by reference and enforces them at runtime.

---

## Boundary with rbx-console

`rbx-console` is the operator UI. It consumes orchestration events to surface:
mission status, session list, context budget utilization, loop termination
events, pending reviews, escalations, and cost. The AOP does not implement the
UI — it produces the events that make the UI meaningful.

---

## Guardrail taxonomy

Guardrails in the AOP operate at four levels. They are enforced at the points
indicated — not as a single centralized check, but as distributed enforcement
throughout the control loop.

### 1. Permission guardrails (Sandbox Gate, Decide stage)

Enforced before any action is executed.

| Guardrail | Enforcement point |
|---|---|
| Tool call blocked if not in `allowed_tools` | Sandbox Gate |
| Repository access blocked if not in `allowed_repositories` | Sandbox Gate |
| Memory access blocked if scope not in `allowed_memory_scopes` | Sandbox Gate |
| Action blocked if in `forbidden_actions` | Sandbox Gate |
| Action blocked if `authority_level` ceiling is insufficient | Sandbox Gate |
| Sub-agent scope must not exceed orchestrator scope | Mission Orchestrator at spawn time |

### 2. Behavioral guardrails (non-overridable, all agents)

Cannot be overridden by any domain contract configuration.

| Guardrail | Enforcement point |
|---|---|
| No agent approves its own output | Review assignment; `guard.self_review_attempted` |
| No push without per-operation operator authorization | Before git push; `harness.permission_check` |
| No deploy without passed human approval gate | Before deploy; `human_approval.required` |
| No mixed commits (docs-only + functional) outside foundation phase | Before commit; `guard.blocked_action` |
| Reviewer `agent_id` ≠ producer `agent_id` | At `review.assigned`; enforced by runtime |
| No unauthorized approval claims in output | Output guardrail; `guard.blocked_action` |

### 3. Context guardrails (Context Management Layer)

Enforced before context is assembled for a model call.

| Guardrail | Enforcement point |
|---|---|
| No secrets or credentials in assembled context | Context assembly; `guard.secret_in_context` |
| Context budget ceiling enforced (default 70%) | Context assembly; `context.budget.*` events |
| Minimum necessary context principle: no more than needed for the step | Context selection logic |
| Memory items outside `allowed_memory_scopes` excluded | Memory retrieval |
| Long-term canonical knowledge verified against current state before use | Memory access; warning if stale |

### 4. Output guardrails (before Thalamus post-call validation)

Enforced after the model responds, before the output is acted on.

| Guardrail | Enforcement point |
|---|---|
| Output diff scoped to `allowed_repositories` only | Output validation; `guard.scope_exceeded` |
| No secrets in output payload | Output scan; `guard.secret_in_context` |
| No claims of approvals that lack EventLog evidence | Output validation; `guard.blocked_action` |
| Output type must match step's `expected_output_type` | Postcondition check |

For the full event catalog see `docs/architecture/THALAMUS-INTEGRATION.md`.
For the Domain Contract guards see `docs/architecture/AGENT-DOMAIN-CONTRACT.md`.

---

## Non-goals

- Not an LLM provider SDK. No direct calls to any AI provider.
- Not a workflow job scheduler (Temporal, Prefect, Airflow).
- Not strategic memory. Strategos owns decision history and rationale.
- Not an IDP. Éden scaffolds agents; the AOP governs their runtime behavior.
- Not a data plane. Thalamus routes AI calls; Agentgateway handles transport.

---

## Implementation status

| Component | Status |
|---|---|
| Control loop spec (`runtime/loop/spec.md`) | ✅ Approved |
| Agent manifest schema (`spec/manifest.schema.json`) | ✅ Foundation |
| Thalamus protocol (`spec/protocol.md`) | ✅ Foundation |
| Governance events (`spec/governance.schema.json`) | ✅ Foundation |
| **Agent Domain Contract spec** | ✅ Foundation (this iteration) |
| **Context Management spec** | ✅ Foundation (this iteration) |
| **Mission Execution Graph spec** | ✅ Foundation (this iteration) |
| **Loop Termination spec** | ✅ Foundation (this iteration) |
| **Thalamus Integration spec** | ✅ Foundation (this iteration) |
| **Orchestration schemas** | ✅ Foundation (this iteration) |
| Runtime implementation (Rust) | 🔲 Planned |
| Go SDK | 🔲 Planned |
| TypeScript SDK | 🔲 Planned |
| CLI scaffolding | 🔲 Planned |

---

## Related documents

- `ARCHITECTURE.md` — rbx-harness overall architecture
- `spec/protocol.md` — Thalamus protocol
- `spec/manifest.schema.json` — Agent manifest schema
- `docs/architecture/AGENT-DOMAIN-CONTRACT.md`
- `docs/architecture/CONTEXT-MANAGEMENT.md`
- `docs/architecture/MISSION-EXECUTION-GRAPH.md`
- `docs/architecture/LOOP-TERMINATION.md`
- `docs/architecture/THALAMUS-INTEGRATION.md`
- `thalamus-core/docs/adr/ADR-0001-thalamus-as-semantic-control-layer.md`
