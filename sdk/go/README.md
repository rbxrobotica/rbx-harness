# rbx-harness Go SDK

**Status**: Not yet implemented

Go SDK for the RBX agent harness. Provides Thalamus protocol client and manifest validation for Go-based agents and services.

---

## Intended usage

```go
import harness "github.com/rbxrobotica/rbx-harness/sdk/go"

// Load and validate agent manifest
manifest, err := harness.LoadManifest("manifest.yaml")
if err != nil {
    log.Fatal(err)
}

// Create a Thalamus client
client, err := harness.NewThalamusClient(harness.Config{
    Endpoint:       "https://thalamus.rbxsystems.com",
    AgentID:        manifest.ID,
    AgentVersion:   manifest.Version,
    Authentication: harness.AuthThalamausManaged,
})

// Send a request to another agent
resp, err := client.Send(ctx, harness.Request{
    Recipient: "strategos-risk-v1",
    Type:      "risk_analysis",
    Data:      payload,
    Priority:  harness.PriorityNormal,
})
```

---

## Target consumers

- `strategos-core` (strategic orchestration engine)
- `thalamus-core` (mediation layer implementation)
- Any Go service that needs to communicate with RBX agents
