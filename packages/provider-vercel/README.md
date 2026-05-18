# @typenode-ai/provider-vercel

Compiles Typenode IR to Vercel Workflow Development Kit (WDK) — `"use workflow"` and `"use step"` TS source.

**Status:** Scaffold — second compiler target. No implementation yet.

## Will provide

- `compile(ir, opts) → VercelDeployment` — emits TS files with WDK directives
- Maps each Typenode IR op to its Vercel equivalent:
  - `do` → `"use step"` function call
  - `route` → native `if/else if/else`
  - `all` → `Promise.all([...])`
  - `loop` → `for ... of` + step function calls
  - `pause` → `await sleep(...)`
  - `listen` → `defineHook` + `for await` over events
  - `stop` → `throw new FatalError(...)`
  - `output` → `return value`
  - `wf` → sub-workflow invocation

This is the **second** cloud target — proves the IR is portable across vendors. See [docs/architecture.md](../../docs/architecture.md).
