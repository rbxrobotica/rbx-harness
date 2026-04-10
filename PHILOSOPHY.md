# PHILOSOPHY — rbx-harness

---

## Why a harness?

An LLM that calls tools is not an agent. An agent is a system that *reliably* acts on its environment over time, with accountability for what it did and why.

The gap between "LLM with tools" and "production agent" is the harness:

- Who authorized this action?
- What state was the system in when the decision was made?
- If the system crashes, can we reconstruct exactly what happened?
- If a human needs to intervene, at which point does the loop pause?
- How do we prevent one misbehaving agent from cascading into others?

These are not LLM problems. They are *systems* problems. The harness solves them.

---

## Core beliefs

### 1. The manifest is the contract

Every agent must declare, upfront and explicitly:
- What it can do
- What it cannot do
- When a human must review its output
- What resources it is allowed to consume

An agent that cannot be described by a manifest is not ready to operate.

### 2. Limitations are as important as capabilities

It is tempting to focus only on what an agent can do. The discipline of this framework is to give equal weight to what it *cannot* do. This is not a weakness to hide — it is a commitment to the system's users and operators.

### 3. Audit is non-negotiable

There is no configuration option to disable audit logging. Every execution cycle produces an immutable record. This is not overhead — it is what makes crash recovery, debugging, and accountability possible.

### 4. The loop owns the execution

No component outside the runtime may initiate, pause, or interrupt an execution cycle without going through the loop's defined mechanisms (pause command, circuit breaker). This single-owner principle is what makes the system predictable.

### 5. Determinism is a design goal, not an accident

Observe, Interpret, Decide, and Evaluate are designed to be pure functions with zero side effects. This means:
- They can be replayed deterministically from event history
- They can be unit-tested without mocks or network calls
- Bugs in these stages reproduce reliably

Act is the only non-deterministic stage. This is the only place where external systems are called. Keeping non-determinism isolated to one stage is a deliberate architecture choice.

### 6. Human review is a first-class primitive

The framework does not treat human-in-the-loop as an afterthought or an edge case. It is modeled explicitly in the manifest's `governance.human_review_required` section. The runtime evaluates these conditions at runtime. Governance is infrastructure, not policy documents.

### 7. The spec precedes the implementation

We define the contract before we write the code. A schema that can be validated independently of any runtime is worth more than a runtime with no documented interface. This is why the `spec/` directory is the first thing this repository ships.

---

## What the harness is not

- It is not an LLM provider SDK
- It is not an agent framework (like LangChain or AutoGen)
- It is not a workflow orchestrator (like Temporal or Prefect)
- It is not domain-specific (it has no knowledge of trading, strategy, or any RBX business domain)

The harness is plumbing. The intelligence lives in the agents that run on top of it.

---

## On language choice

The reference runtime is in Rust. This is not tribal. Rust's ownership model enforces at compile time many of the invariants that the harness cares about:

- No shared mutable state without explicit synchronization
- Explicit error handling (no hidden exceptions propagating through the loop)
- Memory safety without a garbage collector (important for low-latency stage budgets)

The harness spec is language-agnostic. Agents written in Go, TypeScript, Python, or any language that implements the Thalamus protocol are first-class citizens. The runtime is the reference; the spec is the standard.
