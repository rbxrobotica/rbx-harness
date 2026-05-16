# RBX Harness — Thalamus Protocol

**Version**: 1.0 (draft)  
**Status**: Foundation

---

> **Definition note (2026-05-16).** Thalamus is the **semantic control layer
> for AI traffic** (canonical: `thalamus-core`,
> `docs/adr/ADR-0001-thalamus-as-semantic-control-layer.md`). This protocol is
> the agent-facing contract with that control plane. Thalamus decides
> (pre-call) and validates (post-call); it is not a proxy, a gateway, or
> Agentgateway. The actual transport is the replaceable data plane below
> Thalamus. The envelope, `trace_id`, governance block, human-review deferral,
> and OTLP integration below are all consistent with the control-plane model.

## Overview

The Thalamus Protocol defines how agents communicate with Thalamus, the
semantic control layer for AI traffic. It is transport-agnostic at the envelope
level (HTTP/2 + JSON is the reference implementation) but the message format is
the contract.

All agent-to-harness and agent-to-agent communication MUST go through Thalamus,
which governs it (policy, context authorization, validation, audit). Direct
agent-to-agent calls outside this protocol are not permitted.

---

## Message Envelope

Every message — request or response — is wrapped in a standard envelope:

```json
{
  "envelope": {
    "message_id": "<ULID>",
    "protocol_version": "1.0",
    "timestamp": "<RFC3339>",
    "sender": {
      "agent_id": "<agent-id>",
      "agent_version": "<semver>"
    },
    "recipient": {
      "agent_id": "<agent-id>",
      "agent_version": "<semver or 'latest'>"
    },
    "correlation_id": "<ULID or null>",
    "trace_id": "<ULID>"
  },
  "payload": { ... }
}
```

Fields:

| Field | Required | Description |
|---|---|---|
| `message_id` | Yes | ULID — unique per message, monotonically sortable |
| `protocol_version` | Yes | Must match the agent's `interface.protocol_version` |
| `timestamp` | Yes | RFC3339 UTC timestamp |
| `sender.agent_id` | Yes | Must match a registered, active agent ID |
| `recipient.agent_id` | Yes | Target agent ID |
| `correlation_id` | No | Links response to the originating request |
| `trace_id` | Yes | OpenTelemetry trace ID for distributed tracing |

---

## Request lifecycle

```
Sender                  Thalamus                    Recipient
  │                        │                            │
  │──── Request ──────────>│                            │
  │                        │── validate envelope        │
  │                        │── check sender active      │
  │                        │── check recipient active   │
  │                        │── check rate limit         │
  │                        │── route ─────────────────>│
  │                        │                            │── validate payload
  │                        │                            │── execute (harness loop)
  │                        │<─── Response ─────────────│
  │                        │── validate response        │
  │<──── Response ─────────│                            │
  │                        │                            │
```

Thalamus validates the envelope in both directions. The harness runtime validates the payload against the agent's declared request/response schemas.

---

## Request message

```json
{
  "envelope": { "...": "..." },
  "payload": {
    "request_type": "<string>",
    "data": { "...": "..." },
    "context": {
      "session_id": "<ULID or null>",
      "priority": "normal | high | critical",
      "budget": {
        "max_tokens": 4096,
        "timeout_seconds": 30
      }
    }
  }
}
```

The `data` field must validate against the schema declared in `manifest.interface.request_schema`.

---

## Response message

```json
{
  "envelope": {
    "...": "...",
    "correlation_id": "<request message_id>"
  },
  "payload": {
    "status": "success | error | deferred",
    "data": { "...": "..." },
    "error": {
      "code": "<error_code>",
      "message": "<human-readable>",
      "retryable": true
    },
    "governance": {
      "review_id": "<ULID or null>",
      "human_review_required": false,
      "review_sla": null
    },
    "usage": {
      "tokens_consumed": 1024,
      "duration_ms": 350,
      "cycle_id": "<ULID>"
    }
  }
}
```

When `status` is `deferred`, a human review has been triggered. The `governance.review_id` tracks the review. The requester must poll or subscribe for the review outcome before the response is final.

---

## Error codes

| Code | Meaning | Retryable |
|---|---|---|
| `AGENT_NOT_FOUND` | Recipient agent not registered | No |
| `AGENT_SUSPENDED` | Recipient agent is suspended | No |
| `SCHEMA_INVALID` | Payload does not match declared schema | No |
| `RATE_LIMIT_EXCEEDED` | Sender exceeded rate limit | Yes (after backoff) |
| `TIMEOUT` | Agent did not respond within `timeout_seconds` | Yes |
| `CIRCUIT_BREAKER_OPEN` | Recipient's circuit breaker is active | Yes (after reset) |
| `PERMISSION_DENIED` | Action blocked by sandbox rules | No |
| `REVIEW_PENDING` | Human review required before completion | No (await review) |
| `BUDGET_EXCEEDED` | Token or compute budget exceeded | No |
| `PROTOCOL_VERSION_MISMATCH` | Sender and recipient protocol versions incompatible | No |

---

## Signal priorities

Thalamus routes messages according to priority:

| Priority | Use case | Queue behavior |
|---|---|---|
| `critical` | Circuit breaker, operator halt | Preempts current cycle immediately |
| `high` | Operator commands, risk alerts | Front of queue |
| `normal` | Standard requests, signals | FIFO |
| `low` | Housekeeping, health checks | FIFO, droppable when queue is full |

---

## Authentication

All messages are authenticated at the Thalamus layer. Three modes are supported, declared per-agent in `manifest.interface.authentication`:

| Mode | Description |
|---|---|
| `thalamus-managed` | Thalamus issues and validates session tokens. Default mode. |
| `mutual-tls` | mTLS between agent and Thalamus. For high-security executor agents. |
| `api-key` | Static API key. For external integrations only. |

---

## Versioning

The protocol version is `MAJOR.MINOR`:

- `MAJOR` bump: breaking changes (new required fields, removed fields, semantic changes)
- `MINOR` bump: additive changes (new optional fields, new error codes)

An agent declares which protocol version it supports in `manifest.interface.protocol_version`. Thalamus rejects connections from agents whose protocol version is incompatible with the current version.

Compatibility policy: Thalamus supports the current `MAJOR` version and one prior `MAJOR` version. Agents on `0.x` or `N-2` must be upgraded.

---

## OpenTelemetry integration

Every message carries a `trace_id`. The harness runtime creates a child span per execution cycle. Stage durations are recorded as span attributes:

```
trace: <trace_id>
  span: thalamus.route (sender → recipient)
  span: harness.cycle.<cycle_id>
    span: harness.stage.observe
    span: harness.stage.interpret
    span: harness.stage.decide
    span: harness.stage.act
    span: harness.stage.evaluate
    span: harness.stage.persist
```

Exporters: OTLP (gRPC). Compatible with Jaeger, Tempo, Honeycomb.
