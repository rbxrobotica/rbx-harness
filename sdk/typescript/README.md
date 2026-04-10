# rbx-harness TypeScript SDK

**Status**: Not yet implemented

TypeScript SDK for the RBX agent harness. Provides Thalamus protocol client and manifest validation for TypeScript-based agents and frontend services.

---

## Intended usage

```typescript
import { ThalamusClient, loadManifest } from "@rbxrobotica/rbx-harness";

// Load and validate agent manifest
const manifest = await loadManifest("./manifest.yaml");

// Create a Thalamus client
const client = new ThalamusClient({
  endpoint: "https://thalamus.rbxsystems.com",
  agentId: manifest.id,
  agentVersion: manifest.version,
  authentication: "thalamus-managed",
});

// Send a request
const response = await client.send({
  recipient: "strategos-risk-v1",
  type: "risk_analysis",
  data: payload,
  priority: "normal",
});
```

---

## Target consumers

- `rbx-catalog-console` (agent registry UI — manifest viewer and validator)
- Frontend-side agents (browser or Cloudflare Workers)
- `eden` IDP flows that interact with agents
