# CONTRIBUTING — rbx-harness

---

## How this repository is organized

| Directory | What it is | Who changes it |
|---|---|---|
| `spec/` | Language-agnostic contracts | Spec review required for any change |
| `runtime/` | Rust execution engine | Runtime team |
| `sdk/go/` | Go language bindings | Platform team |
| `sdk/typescript/` | TypeScript language bindings | Platform team |
| `cli/` | Developer CLI | Platform team |

---

## Proposing a spec change

Changes to `spec/` are the most consequential — they affect every agent in the RBX ecosystem. Before opening a PR:

1. Open an issue describing the problem the change solves
2. Include which existing agents would be affected
3. Indicate whether the change is additive (non-breaking) or structural (breaking)

Breaking changes require:
- Major version bump in the schema `$id`
- Migration guide in `CHANGELOG.md`
- Updated examples in `spec/examples/`

Additive changes (new optional fields) do not require a version bump.

---

## Commit conventions

We use [Conventional Commits](https://www.conventionalcommits.org/):

```
feat: add max_context_tokens field to manifest resources
fix: correct governance.audit_level enum values
spec: add governance.schema.json initial draft
docs: clarify Thalamus protocol request envelope
chore: update schema $id to v0.2
```

Use `spec:` for changes under `spec/`. Use `feat:` for new runtime or SDK functionality.

---

## Validation before committing

When modifying `spec/manifest.schema.json` or `spec/governance.schema.json`:

1. Validate the schema itself is valid JSON Schema draft-07
2. Validate all examples under `spec/examples/` still pass
3. Run `rbx-harness validate manifest` against the example agent manifest (once CLI is available)

---

## Versioning

This repository follows [Semantic Versioning](https://semver.org/):

- **MAJOR**: Breaking spec changes
- **MINOR**: New capabilities (new optional fields, new SDK features)
- **PATCH**: Bug fixes, documentation corrections

The spec version and the runtime/SDK versions are independent. A MAJOR spec change does not require simultaneous runtime changes — but runtime changes that implement a new spec version must bump their minor version.

---

## Opening a PR

- Target the `main` branch
- Keep PRs focused (one concern per PR)
- Reference the issue number in the PR description
- Ensure CI passes before requesting review

---

## Questions

For architectural questions, open a GitHub Discussion. For bugs, open an issue with reproduction steps.
