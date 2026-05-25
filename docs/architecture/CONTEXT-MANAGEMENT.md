# Context Management

**Status**: Foundation
**Version**: 0.1

---

## Overview

Context management is the discipline of deciding **what enters a model call,
at what cost, and why**. In an orchestrated agent system, this is not an
implementation detail — it is a first-class architectural concern that
determines correctness, cost, and reliability.

This document defines three layers:

1. **LLM Context Window** — the hard technical limit
2. **Agent Working Context** — the operational state of an agent during a mission
3. **Context Management Layer** — what controls what enters the window

It also defines the **Memory Taxonomy** and the **Context Budget Policy**.

---

## 1. LLM Context Window

The LLM Context Window is the total token budget available for a single model
call. It belongs to the **model**, not the agent.

Everything sent to the model in a single call counts against this window:

| Slot | Examples |
|---|---|
| System prompt | Harness instructions, governance rules |
| Role prompt | Agent identity, domain constraints |
| Tool definitions | All tools exposed to the model in this call |
| Governance instructions | Policies, guards, review requirements |
| Conversation history | Prior exchanges in the session |
| Selected files | Source files, diffs, ADRs |
| Tool outputs | Results from prior tool calls |
| Logs | Truncated logs passed as context |
| Diffs | Code or doc diffs under review |
| Plans | Execution plans, steps |
| Summaries | Compressed prior research |
| Test outputs | Test run results |
| Expected completion | Budget reserved for the model's response |

The window is finite. Every token admitted is a tradeoff. The Context
Management Layer is the gating mechanism.

---

## 2. Agent Working Context

The Agent Working Context is the **operational state of the agent during a
mission**. It is distinct from the LLM Context Window: the working context is
the full picture of what the agent knows and is doing; the window is the subset
that fits into a single model call.

The working context is persisted in the EventLog and in session memory. It is
not ephemeral.

### Required fields (when applicable)

| Field | Description |
|---|---|
| `mission_id` | Which mission this context belongs to |
| `session_id` | Current session |
| `objective` | What the agent is trying to accomplish |
| `scope` | What is in scope (repositories, files, systems) |
| `constraints` | Hard constraints (budget, time, forbidden actions) |
| `selected_sources` | Files, ADRs, schemas currently selected |
| `allowed_tools` | Tools available in this session |
| `current_plan` | PromptExecutionStep list for the current step |
| `completed_steps` | Steps already finished in this session |
| `intermediate_outputs` | Outputs produced but not yet finalized |
| `risks_encountered` | Detected risks or blockers |
| `decisions_made` | Key decisions taken during the mission |
| `pending_items` | Open items, blocked steps, awaiting review |
| `next_recommended_step` | What the agent proposes to do next |

The working context is what the Context Management Layer selects from when
building the window for the next model call. Not all of it goes in every call.

---

## 3. Context Management Layer

The Context Management Layer is the component that decides what enters the
model call. It is explicit and governed — not a hidden implementation detail.

### Responsibilities

1. **Budget allocation** — Enforce the context budget policy (see §4).
2. **Slot assignment** — Assign token budget to each input type.
3. **Selection** — Choose which files, history turns, logs, and tool outputs to include.
4. **Compression** — Summarize or compress inputs that exceed their slot budget.
5. **Memory retrieval** — Fetch relevant short/medium-term memory items by scope and relevance.
6. **Noise exclusion** — Drop irrelevant logs, verbose outputs, and out-of-scope history.
7. **Source separation** — Never conflate raw context, summaries, and canonical references.

### Context guardrails (applied during assembly)

Before the assembled context is sent to the model, two hard guardrails apply:

**1. No secrets in context** — The Context Management Layer scans assembled
context for credential patterns (API keys, tokens, passwords, private keys).
If detected: discard the offending item, emit `guard.secret_in_context`, and
proceed without it. If the secret is in a required slot (e.g., system prompt),
halt and escalate — never send.

Secrets must never enter the model call. Agents needing to work with
credentials should receive references to them (e.g., env var names, secret
path IDs) not the values themselves.

**2. Minimum necessary context** — A step receives only the context required
for its declared inputs (`input_context_refs`). The Context Management Layer
must not load additional files, history, or memory items "just in case." Each
extra token is a potential scope-creep vector and a budget cost. Unused context
items are excluded even if available.

This principle applies to tool definitions too: only tools that may be called
in this step should be included in the tool definitions slot.

### Selection priority (high to low)

1. Governance instructions and hard constraints (never compressed)
2. Current step preconditions and postconditions
3. System prompt and role prompt (never compressed)
4. Tool definitions for tools needed in this step
5. Current plan and immediate prior step outputs
6. Canonical references (ADRs, schemas) relevant to current step
7. Medium-term memory items with high confidence
8. Short-term memory items relevant to current step
9. Conversation history (truncated oldest-first if over budget)
10. Logs, diffs, test outputs (truncated to slot budget)

---

## 4. Context Budget Policy

The Context Budget Policy enforces that model calls stay within a safe
utilization ceiling, leaving margin for the model's response and unexpected
growth.

**Default policy**: No model call may exceed **70% of the model's declared
context window** in input tokens, except when a logged justification event is
emitted.

The remaining 30% is reserved for:
- Model response
- Additional tool calls triggered mid-response
- Unexpected log or error growth
- Retry context if the first attempt fails

### Budget line items

Each session carries a `ContextBudgetPolicy` that declares a token allocation
for each input type. The allocations sum to at most `ceiling_ratio × total_window`.

| Line item | Default share of ceiling | Notes |
|---|---|---|
| `system_prompt` | Fixed (declared in manifest) | Never compressed |
| `role_prompt` | Fixed (declared in manifest) | Never compressed |
| `tool_definitions` | Variable (count × avg_per_tool) | Drop unused tools |
| `governance_instructions` | Fixed | Never compressed |
| `conversation_history` | ≤15% | Truncate oldest first |
| `selected_files` | ≤20% | Summarize if over limit |
| `tool_outputs` | ≤10% | Truncate verbose outputs |
| `logs` | ≤5% | Hard limit; never exceed |
| `diffs` | ≤10% | Split across calls if over limit |
| `plans` | ≤5% | Compress to step summaries |
| `summaries` | ≤5% | From prior summarization sessions |
| `test_outputs` | ≤5% | Truncate to failures + first N lines |
| `completion_budget` | Fixed 30% | Reserved, never allocated to input |

### Budget estimation

The initial implementation may use character-count estimation (÷ 4 for
approximate tokens). Future versions should use provider-specific tokenizers.
The policy schema (`spec/context-budget.schema.json`) includes a
`tokenizer_hint` field for this migration path.

### Budget violation handling

If the assembled context would exceed the ceiling:

1. Apply compression to compressible slots (conversation_history, logs, diffs,
   tool_outputs) in reverse priority order until within budget.
2. If still over budget after compression: emit
   `context.budget.compression_applied` event and proceed with compressed context.
3. If compression is insufficient: emit `context.budget.exceeded` event, halt
   the step, and escalate to the Mission Orchestrator.
4. The orchestrator may: split the step across two sessions, open a
   summarization session, or escalate to human.

A step must never silently truncate input without emitting a budget event.

---

## 5. Memory Taxonomy

Agent memory is organized into four layers by lifetime and authority.

### Layer 1: Ephemeral Context

**Scope**: Within a single model call.
**TTL**: Ends with the call.
**Authority**: Zero — ephemeral context is not persisted or referenced.
**Use**: Intermediate reasoning, chain-of-thought, working notes within a step.

### Layer 2: Short-Term Memory

**Scope**: Current task or session.
**TTL**: Minutes to hours; expires when session closes or TTL elapses.
**Authority**: Working — reliable within the session; not canonical.
**Use**: Step outputs, discovered risks, intermediate decisions, working plans.

### Layer 3: Medium-Term Memory

**Scope**: Project, epic, or active initiative.
**TTL**: Days to weeks; expires when the initiative closes.
**Authority**: Working — subject to revision; not yet validated.
**Use**: Mission state, project-level discoveries, cross-session context.

### Layer 4: Long-Term Canonical Knowledge

**Scope**: Institutional; across all projects.
**TTL**: Permanent until explicitly deprecated.
**Authority**: Canonical — source of truth.
**Use**: ADRs, approved schemas, decision registries, architecture contracts.

### Memory item metadata

Every persisted memory item (layers 2–4) carries:

| Field | Type | Description |
|---|---|---|
| `memory_id` | ULID | Unique identifier |
| `type` | enum | short_term / medium_term / long_term_canonical |
| `source` | string | Where this memory came from (agent_id, tool, document) |
| `scope` | string | Project, mission, or global |
| `owner` | string | agent_id or operator responsible |
| `confidence` | float [0,1] | Estimated reliability of this memory |
| `created_at` | RFC3339 | Creation timestamp |
| `expires_at` | RFC3339 or null | Expiry (null = permanent) |
| `canonical_reference` | string or null | URL/path to the authoritative source, if any |
| `status` | enum | draft / working / validated / canonical / deprecated |
| `content` | object | Memory payload (type-specific) |

### Critical invariant

**Memory aids agents in working; it is not the source of canonical truth.**

An agent acting on a medium-term memory item must verify against the canonical
source (repository state, ADR, schema) before making decisions that would
affect the repository or external systems. A stale memory item does not justify
a stale action.

### Memory and Thalamus

Memory reads and writes are emitted as `memory.read` and `memory.write`
orchestration events. Thalamus can observe memory access patterns for
audit and policy enforcement purposes.

---

## Related documents

- `spec/context-budget.schema.json` — Context Budget Policy schema
- `spec/memory.schema.json` — Memory item schema
- `spec/session.schema.json` — Session types and their context constraints
- `spec/mission-execution.schema.json` — Agent Working Context fields
- `docs/architecture/MISSION-EXECUTION-GRAPH.md` — How context is consumed per step
- `docs/architecture/THALAMUS-INTEGRATION.md` — Context budget events
