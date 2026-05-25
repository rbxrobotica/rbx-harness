# Agent Domain Contract

**Status**: Foundation
**Version**: 0.1

---

## Overview

The Agent Domain Contract defines the **execution scope and runtime authority**
of an agent for a given mission or role. It extends the Agent Manifest
(`spec/manifest.schema.json`) with permissions that are evaluated at mission
and session level — not just at manifest validation time.

The manifest declares *what an agent can do in principle*.
The Domain Contract declares *what this agent is authorized to do in this context*.

Both are enforced. The manifest is static (validated at registration). The
Domain Contract is resolved at mission creation and re-evaluated at each
Sandbox Gate (control loop stage 3, Decide).

See `spec/domain-contract.schema.json` for the machine-readable schema.

---

## Domain taxonomy

| Domain | Scope |
|---|---|
| `code` | Source files, tests, build outputs |
| `architecture` | ADRs, design documents, proposals |
| `governance` | Policies, decision records, compliance checks |
| `commercial` | Sales, pricing, contracts, messaging |
| `operations` | Infrastructure, deployment, monitoring |
| `support` | User-facing responses, incident triage |
| `research` | Exploration, synthesis, analysis |
| `writing` | Documentation, guides, changelogs |
| `testing` | Test design, execution, review |
| `deployment_readiness` | Pre-deploy gates, checklist evaluation |

An agent must not accept a session outside its declared domain unless explicitly
authorized by a Mission Orchestrator with `authority_level: orchestrator`.

---

## Authority levels

| Level | Description |
|---|---|
| `observer` | Read-only. No state-mutating tool calls |
| `advisor` | Produces recommendations; cannot execute them |
| `executor` | Executes within scope (edit files, run tests) |
| `orchestrator` | Spawns and coordinates sub-agents |
| `governor` | Evaluates, validates, and can veto outputs |

Authority level is the ceiling for all other permissions. An `observer` with
`can_modify_code: true` in the schema would still be blocked — the runtime
resolves effective permission as `min(authority_level_ceiling, declared_field)`.

---

## Contract fields (summary)

Full schema: `spec/domain-contract.schema.json`

| Field | Type | Description |
|---|---|---|
| `agent_id` | string | Must match `manifest.id` |
| `domain` | enum | Primary domain |
| `allowed_repositories` | string[] | Git repos this agent may access |
| `allowed_tools` | string[] | Tool IDs this agent may invoke |
| `allowed_memory_scopes` | enum[] | ephemeral, short_term, medium_term, long_term_canonical |
| `allowed_session_types` | enum[] | Session types this agent may operate in |
| `authority_level` | enum | observer / advisor / executor / orchestrator / governor |
| `can_orchestrate_subagents` | bool | May spawn and coordinate sub-agents |
| `can_modify_code` | bool | May edit source files |
| `can_create_commits` | bool | May commit changes locally |
| `can_push` | bool | May push to remote (requires `can_create_commits: true`) |
| `can_deploy` | bool | May trigger deployment pipelines |
| `requires_human_approval_for` | string[] | Action types that always require a human gate |
| `output_types` | string[] | Expected output types for this domain |
| `forbidden_actions` | string[] | Actions explicitly prohibited; not delegatable |
| `review_requirements` | object | Who must review before output is acted on |

---

## Guards (always enforced, not overridable)

These guards apply to every agent, regardless of domain contract configuration.
Violating them produces a `harness.permission_check { decision: denied }` event.

1. **No self-review.** An agent must not approve, merge, or confirm its own output.
2. **No push without per-operation authorization.** `can_push: true` in the
   contract is necessary but not sufficient. Each push requires explicit
   authorization in the current operator message. Blanket allowlists are not
   per-push authorization.
3. **No deploy without human approval gate.** `can_deploy: true` requires a
   passed `requires_human_approval_for: [deploy]` gate before deployment proceeds.
4. **No mixed commits.** Docs-only changes and functional changes must not be
   in the same commit unless the repository is in its declared foundation phase.
5. **No out-of-scope memory.** An agent must not read or write memory outside
   its `allowed_memory_scopes`.
6. **No out-of-scope tools.** An agent must not invoke tools outside
   `allowed_tools`.
7. **No context budget overrun.** An agent must not initiate a step that would
   cause the session to exceed its `context_budget_policy.ceiling_ratio` without
   a logged justification event.

These guards are evaluated in the **Decide** stage (Sandbox Gate sub-stage).
Guard violations do not terminate the mission — they produce a blocked
`EngineAction` and a `guard.blocked_action` orchestration event, then the
step moves to `blocked` status awaiting escalation or operator intervention.

---

## Sub-agent scope non-escalation

When a Mission Orchestrator spawns a sub-agent, the sub-agent's effective
permissions are the **intersection** of the orchestrator's domain contract and
the sub-agent's own domain contract. Permissions only narrow down the hierarchy
— they never expand.

This is enforced at spawn time by the Mission Orchestrator, not by the
sub-agent itself:

```
orchestrator.allowed_tools = [read_file, edit_file, run_tests]
sub_agent.allowed_tools    = [read_file, edit_file, run_tests, deploy]

effective_tools = intersection = [read_file, edit_file, run_tests]
                                  ↑ deploy is stripped
```

If the Mission Orchestrator does not have `can_push: true`, no sub-agent it
spawns may push, regardless of the sub-agent's own contract. The orchestrator
cannot delegate authority it does not have.

This invariant is logged as `mission.step.started { inherited_scope: true }`
and violations emit `guard.permission_escalation_attempted`.

---

## Reviewer independence invariant

A Reviewer Agent evaluating an output must satisfy all three conditions:

1. `reviewer.agent_id ≠ producer.agent_id`
2. The reviewer was not a co-executor of any prior step in the same diff's
   scope (i.e., did not produce any of the input_context_refs for this output)
3. The reviewer has `authority_level: governor` or is an explicit human operator

These conditions are checked by the runtime at `review.assigned` time. If no
eligible reviewer is available, the runtime emits `review.required` and pauses
execution, escalating to the Mission Orchestrator or human.

---

## Relation to manifest governance

The manifest's `governance.human_review_required` conditions are evaluated
per-cycle based on observed state. The Domain Contract's
`requires_human_approval_for` action list is evaluated per-step based on the
intended action type. Both are evaluated independently; either can trigger a
`governance.review_triggered` event and pause execution.

---

## Review requirements

The `review_requirements` block declares which output types require review,
by whom, and with what minimum reviewer count before an output can be acted on:

```yaml
review_requirements:
  code_diff:
    reviewer_domains: [architecture, governance]
    min_reviewers: 1
    blocking: true
  architecture_proposal:
    reviewer_domains: [governance]
    min_reviewers: 1
    blocking: true
  deployment_manifest:
    reviewer_domains: [operations, governance]
    min_reviewers: 2
    blocking: true
```

A reviewer must have `authority_level: governor` or be an explicit operator.
An agent with the same `agent_id` as the output producer cannot be the reviewer.

---

## Example: code domain executor

```yaml
agent_id: robson-code-agent-v1
domain: code
allowed_repositories:
  - robson
  - robson-domain
allowed_tools:
  - read_file
  - edit_file
  - run_tests
  - git_status
  - git_diff
allowed_memory_scopes:
  - ephemeral
  - short_term
  - medium_term
allowed_session_types:
  - coding
  - review
  - testing
authority_level: executor
can_orchestrate_subagents: false
can_modify_code: true
can_create_commits: true
can_push: false
can_deploy: false
requires_human_approval_for:
  - git_push
  - deploy
  - merge_to_main
output_types:
  - code_diff
  - test_results
  - review_comments
forbidden_actions:
  - modify_secrets
  - delete_branch
  - force_push
  - approve_own_output
review_requirements:
  code_diff:
    reviewer_domains: [architecture, governance]
    min_reviewers: 1
    blocking: true
```

## Example: architecture advisor

```yaml
agent_id: rbx-arch-advisor-v1
domain: architecture
allowed_repositories:
  - rbx-harness
  - thalamus-core
  - strategos-core
  - rbx-governance
allowed_tools:
  - read_file
  - search_code
  - read_adr
allowed_memory_scopes:
  - ephemeral
  - short_term
  - medium_term
  - long_term_canonical
allowed_session_types:
  - planning
  - architecture
  - research
  - review
authority_level: advisor
can_orchestrate_subagents: false
can_modify_code: false
can_create_commits: false
can_push: false
can_deploy: false
requires_human_approval_for: []
output_types:
  - architecture_proposal
  - adr_draft
  - review_comments
forbidden_actions:
  - modify_code
  - create_commit
  - approve_own_output
review_requirements:
  architecture_proposal:
    reviewer_domains: [governance]
    min_reviewers: 1
    blocking: true
```

---

## Related documents

- `spec/domain-contract.schema.json` — Machine-readable schema
- `spec/manifest.schema.json` — Agent manifest (static capabilities)
- `spec/governance.schema.json` — Governance event schema
- `docs/architecture/AGENT-ORCHESTRATION-PLANE.md` — AOP overview
- `docs/architecture/MISSION-EXECUTION-GRAPH.md` — Where contracts are resolved
