# @typenode-ai/core

Typenode IR (intermediate representation) — the typed JSON document that `@typenode-ai/sdk` synthesizes to and that vendor compilers consume.

**Status:** Scaffold — no implementation yet.

## Will provide

- IR Zod schema (`WorkflowIR`, `IRNode`, `ExprRef`, `CodeRef`, `RetryPolicy`, ...)
- `synthesize(workflowDef)` — runs a workflow build function once at synth time and emits IR
- Handle/Expr proxy implementation (the recording machinery)
- IR validation, diffing, and canonical serialization

Designed so customers never author IR by hand — it's a compiler intermediate. See [docs/architecture.md](../../docs/architecture.md) for the design rationale.
