# @typenode-ai/eslint-plugin

ESLint rules for Typenode workflows. Enforces determinism boundaries and SDK contract.

**Status:** Scaffold — no implementation yet.

## Will provide

- `no-non-deterministic-globals` — bans `Date.now()`, `Math.random()`, `crypto.randomUUID()` inside `step.do(fn)` bodies (use `step.now()`, `step.random()`, `step.uuid()` instead)
- `no-outer-scope-capture-in-expr` — bans mutable variable capture in `step.expr` closures
- `step-do-name-unique` — warns when two `step.do` calls in the same scope share a name
- `no-await-on-handle` — bans `await step.do(...)` at synth time (handles aren't promises)

See [docs/methods-v0.1.md §7.4](../../docs/methods-v0.1.md) for the full rule list.
