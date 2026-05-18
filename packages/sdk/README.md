# @typenode-ai/sdk

The operator surface for Typenode workflows. Re-exports `workflow()` and the `step.*` builder.

**Status:** Scaffold — no implementation yet. See the [main repo README](../../README.md) for status.

## Will provide

The 17-member surface locked in [docs/methods-v0.1.md](../../docs/methods-v0.1.md):

- **Operators:** `step.do`, `step.route`, `step.all`, `step.loop`, `step.pause`, `step.listen`, `step.stop`, `step.output`, `step.wf`
- **Helpers:** `step.input`, `step.secret`, `step.expr`
- **Determinism + Observability:** `step.log`, `step.now`, `step.random`, `step.uuid`, `step.meta`

## Eventual usage

```typescript
import { workflow, step } from "@typenode-ai/sdk";
import { z } from "zod";

export default workflow({
  id: "my-workflow",
  input: z.object({ /* ... */ }),
}, (s) => {
  // build your workflow
});
```

See [docs/architecture.md](../../docs/architecture.md) for the full design.
