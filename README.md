# rbx-harness

The RBX Systems agent harness framework.

rbx-harness is the foundational layer that defines what it means to be an RBX agent — how it declares capabilities, how it executes, how it is governed, and how it integrates with the broader RBX ecosystem (Robson, Strategos, Thalamus, Éden).

---

## What is a harness?

A harness is the scaffolding that allows an AI agent to operate reliably in production. It is not an LLM SDK. It is the layer that solves:

| Problem | What the harness provides |
|---|---|
| Agent does unauthorized things | Permission model + sandboxing |
| Agent loops or stalls | Execution loop with budget and timeout |
| No one knows what the agent did | Immutable audit trail via EventLog |
| Tool called with wrong schema | Tool registry with schema validation |
| Context window exhausted | Context management and compression |
| Human review needed | Governance triggers and approval workflows |
| Failure mid-cycle | Checkpointing and crash recovery |

---

## Architecture overview

```
┌─────────────────────────────────────────────────────────┐
│                     RBX Agent                           │
│                                                         │
│  manifest.yaml  ←  declares capabilities & governance   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │              rbx-harness runtime                │    │
│  │                                                 │    │
│  │  Observe → Interpret → Decide → Act →           │    │
│  │  Evaluate → Persist                             │    │
│  │                                                 │    │
│  │  [sandbox] [audit] [governance] [context]       │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  ← Thalamus protocol (signal mediation)                 │
└─────────────────────────────────────────────────────────┘
```

The harness has three layers:

1. **Spec** — Language-agnostic contracts: manifest schema, protocol definition, governance schema
2. **Runtime** — The execution engine (Rust). Implements the control loop, sandboxing, audit log, context budget
3. **SDK** — Language bindings (Go, TypeScript) that implement the Thalamus protocol and manifest validation

---

## Repository layout

```
rbx-harness/
├── spec/                    # Language-agnostic contracts (source of truth)
│   ├── manifest.schema.json # Agent manifest schema
│   ├── protocol.md          # Agent ↔ harness communication protocol
│   └── governance.schema.json
│
├── runtime/                 # Reference runtime (Rust) — not yet implemented
│   └── loop/
│       └── spec.md          # Control loop specification (from Robson v3 case)
│
├── sdk/
│   ├── go/                  # Go SDK (for Strategos, Thalamus)
│   └── typescript/          # TypeScript SDK (for frontends and edge agents)
│
└── cli/                     # rbx-harness CLI — scaffold, validate, register
```

---

## Design principles

1. **Spec is the contract.** The manifest declares what an agent can and cannot do. If it is not in the manifest, the agent cannot do it.

2. **Runtime owns the loop.** No external component may initiate, pause, or interrupt an execution cycle outside the runtime.

3. **Limitations are first-class.** Declaring what an agent *cannot* do is as important as declaring what it can.

4. **Audit is non-negotiable.** Every cycle produces an immutable event log entry. This is not configurable.

5. **Human review is a first-class primitive.** Governance triggers are declared in the manifest, not hardcoded in agent logic.

6. **Determinism within the cycle.** Observe → Interpret → Decide are pure and deterministic. Act is the only non-deterministic stage.

---

## Case study: Robson v2/v3

The control loop specification in `runtime/loop/spec.md` is derived from [Robson](https://github.com/rbxrobotica/robson)'s v3 architecture — the most mature governed execution system in the RBX portfolio. Robson proved that the `Observe → Interpret → Decide → Act → Evaluate → Persist` pipeline applies beyond trading: it is a general pattern for any agent that must act on external signals with auditability and risk control.

---

## Relation to other RBX products

| Product | Relation to rbx-harness |
|---|---|
| **Robson** | Origin of the control loop spec. Will adopt rbx-harness runtime crate when available |
| **Strategos** | Origin of the manifest and governance schemas. Will register agents via rbx-harness manifest |
| **Thalamus** | The semantic control layer for AI traffic (canonical: `thalamus-core`, ADR-0001). Implements the agent-facing protocol in `spec/protocol.md`; may mediate model payloads inline through BackendPort for enforcement, but does not host inference or own provider transport |
| **Éden** | IDP that uses the CLI to scaffold new agents and register them in the catalog |

---

## Status

This repository is in the **foundation phase**. The spec layer is being stabilized. Runtime and SDK implementation will follow.

See [ARCHITECTURE.md](ARCHITECTURE.md) for the full design. See [CONTRIBUTING.md](CONTRIBUTING.md) to propose changes to the spec.
