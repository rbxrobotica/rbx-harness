# AGENTS.md — rbx-harness

Canonical AI agent instructions for this repository.
If any file in this repository conflicts with this file, this file takes precedence.

---

## What this repository is

rbx-harness is a **framework specification and reference implementation** — not a deployed product. It defines contracts (schemas, protocols) and will house a Rust runtime crate and language SDKs.

When working here, you are either:
- Evolving the **spec** (schemas, protocol, governance contracts)
- Implementing the **runtime** (Rust crate for the control loop)
- Building an **SDK** (Go or TypeScript bindings)
- Building the **CLI** (scaffolding and validation tooling)

---

## Language conventions

| Layer | Language | Rationale |
|---|---|---|
| Spec | JSON Schema, Markdown | Language-agnostic |
| Runtime | Rust | Aligns with Robson v2/v3; determinism and memory safety |
| SDK Go | Go | Aligns with Strategos Core and Thalamus |
| SDK TypeScript | TypeScript | Aligns with frontend agents and edge workers |
| CLI | Rust or Go | TBD at implementation time |

Do not introduce Python as a new dependency. The Python presence in Robson is legacy.

---

## Spec changes

Changes to `spec/manifest.schema.json`, `spec/governance.schema.json`, or `spec/protocol.md` are **breaking changes** if they remove or rename existing fields. Any breaking change requires:

1. A version bump in the schema `$id`
2. A migration note in `CHANGELOG.md`
3. An update to any affected example manifests under `spec/examples/`

Additive changes (new optional fields) are non-breaking.

---

## Naming conventions

- Schema `$id` pattern: `https://rbxsystems.com/schemas/rbx-harness/<version>/<file>`
- Manifest `id` pattern: `<agent-name>-v<major>` (e.g., `robson-v3`)
- Protocol versions: `MAJOR.MINOR` (e.g., `1.0`)
- CLI commands: kebab-case (e.g., `rbx-harness new agent`, `rbx-harness validate manifest`)

---

## Commit policy

- Use conventional commits: `feat:`, `fix:`, `spec:`, `docs:`, `chore:`
- Use `spec:` prefix for changes to the `spec/` directory
- Prefer atomic commits (one concern per commit)
- Keep the worktree clean after push

---

## Relation to other RBX repositories

When referencing other RBX repositories in documentation or code:
- Robson control loop: source is `docs/architecture/v3-control-loop.md` in the robson repo
- Strategos agent manifest: source is `schema/agent-manifest.schema.json` in the strategos-agents repo
- Do not copy and re-embed content from other repos — cross-reference instead

---

## What NOT to do

- Do not implement domain-specific logic (trading rules, risk models) in this framework
- Do not couple the runtime to any specific LLM provider
- Do not add governance bypass mechanisms ("skip audit", "disable permission check")
- Do not make the manifest optional — every RBX agent must have one
