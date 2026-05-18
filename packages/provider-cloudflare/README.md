# @typenode-ai/provider-cloudflare

Compiles Typenode IR to Cloudflare Workflows TS + a WFP dispatch Worker (leaf code execution).

**Status:** Scaffold — first compiler target. No implementation yet.

## Will provide

- `compile(ir, opts) → CloudflareDeployment` — emits CF Workflow class + WFP Worker script
- Maps each Typenode IR op to its CF equivalent:
  - `do` → `await step.do(name, fn)`
  - `route` → native `if/else if/else`
  - `all` → `Promise.all([...])`
  - `loop` → `for ... of` + step calls
  - `pause` → `step.sleep`
  - `listen` → `step.waitForEvent`
  - `stop` → `throw new NonRetryableError(...)`
  - `output` → `return value`
  - `wf` → sub-workflow invocation

Reuses the existing pattern from `typenode-api/packages/shared/src/wfp/script-compiler.ts` (private to Typenode's platform).
