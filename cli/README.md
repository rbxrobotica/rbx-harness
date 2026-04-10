# rbx-harness CLI

**Status**: Not yet implemented  
**Binary name**: `rbx-harness`

Developer tooling for the RBX agent harness. Used to scaffold new agents, validate manifests, and register agents with the RBX catalog (Éden integration).

---

## Planned commands

```bash
# Scaffold a new agent directory with manifest template
rbx-harness new agent <name> --product <robson|strategos|thalamus|truthmetal|platform>

# Validate a manifest against the spec schema
rbx-harness validate manifest [--file manifest.yaml]

# Check governance triggers for completeness
rbx-harness validate governance [--file manifest.yaml]

# Show breaking changes between two manifest versions
rbx-harness diff <manifest-v1.yaml> <manifest-v2.yaml>

# Register an agent in the RBX catalog (Éden integration)
rbx-harness register [--file manifest.yaml] [--env staging|production]

# List agents registered in the catalog
rbx-harness list agents [--product <name>] [--status active]
```

---

## `new agent` scaffold output

Running `rbx-harness new agent my-agent --product strategos` produces:

```
agents/my-agent/
├── manifest.yaml          ← populated template with all required fields
├── README.md              ← agent documentation template
├── schemas/
│   ├── request.schema.json
│   └── response.schema.json
└── tests/
    └── manifest_test.yaml ← example request/response for validation
```

---

## Éden integration

The CLI is the primary integration point with [Éden](https://github.com/rbxrobotica/eden), the RBX Internal Development Platform. When Éden provisions a new agent:

```
Éden: "create new agent"
  → calls: rbx-harness new agent <name> --product <product>
  → scaffolds directory
  → calls: rbx-harness validate manifest
  → calls: rbx-harness register --env staging
  → provisions infra via rbx-infra (Kustomize)
  → registers in rbx-catalog-registry
```

---

## Language decision

The CLI language (Rust or Go) will be decided at implementation time. Both are viable:
- Rust: ships as a single static binary, consistent with the runtime crate
- Go: consistent with Strategos tooling, simpler cross-compilation for the Éden IDP server
