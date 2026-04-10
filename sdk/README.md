# rbx-harness SDKs

Language-specific implementations of the Thalamus protocol and manifest validation utilities.

---

## Available SDKs

| SDK | Directory | Language | Target |
|---|---|---|---|
| Go SDK | `sdk/go/` | Go | Strategos Core, Thalamus, Go-based agents |
| TypeScript SDK | `sdk/typescript/` | TypeScript | Frontend agents, edge workers, rbx-catalog-console |

---

## What each SDK provides

All SDKs implement the same surface area:

1. **Manifest loading and validation** — parse `manifest.yaml`, validate against `spec/manifest.schema.json`
2. **Thalamus client** — send and receive messages using the protocol defined in `spec/protocol.md`
3. **Envelope construction** — build valid message envelopes (ULID generation, timestamp, trace propagation)
4. **Error handling** — typed error codes from the protocol spec
5. **Context budget tracking** — track token consumption against `manifest.resources.tokens.max_per_request`

---

## What SDKs do NOT provide

- The execution loop (that is the runtime's responsibility)
- Domain-specific logic (trading, strategy, etc.)
- LLM provider clients (agents bring their own)

---

## Status

Both SDKs are not yet implemented. They will follow runtime implementation.
