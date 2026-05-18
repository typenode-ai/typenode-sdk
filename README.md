# Typenode SDK

> Cloud-agnostic workflows for the agent era. Write once in TypeScript, run on Cloudflare, Vercel, AWS, or your own infrastructure.

**Status:** Pre-release — design phase. The architecture and method surface are locked at v0.1; implementation is in progress. See [`docs/`](./docs/) for the design memos.

---

## What this is

Typenode SDK is a TypeScript builder DSL for defining durable, agentic workflows. You write a `.workflow.ts` file using a small set of operators (`step.do`, `step.route`, `step.all`, `step.loop`, `step.pause`, `step.listen`, ...) and the SDK synthesizes a portable intermediate representation. Vendor-specific compilers translate the IR into Cloudflare Workflows, Vercel WDK, AWS Step Functions, Inngest, or a self-hosted Node runner.

Three properties make this useful:

1. **One canonical, multiple targets.** Your `.workflow.ts` file is the source of truth. Switching clouds is a compiler flag, not a rewrite.
2. **Human + agent co-authored.** Engineers edit the TS directly. AI agents emit patches against the same file. The visual canvas reads from the derived IR. All surfaces converge on one artifact.
3. **You own the code.** Synthesize locally, commit to your repo, deploy to your infrastructure. If Typenode disappears tomorrow, your workflows keep running.

## Quick taste

```typescript
import { workflow, step } from "@typenode-ai/sdk";
import { z } from "zod";

export default workflow({
  id: "refund-handler",
  input: z.object({ orderId: z.string() }),
}, (s) => {
  const order = step.do("fetch-order", {
    deps: [s.input.orderId],
    fn: async ([orderId]) => fetchOrder(orderId),
  });

  return step.route({
    cases: [{
      name: "eligible",
      when: step.expr([order], ([o]) => o.refundable),
      then: (s) => step.do("issue-refund", {
        deps: [order],
        fn: async ([o]) => stripe.refunds.create({ charge: o.chargeId }),
      }),
    }],
    default: (s) => step.stop("refund-window-expired"),
  });
});
```

Deploy to Cloudflare:

```bash
npx @typenode-ai/cli deploy refund-handler.workflow.ts --target=cloudflare
```

Switch to Vercel:

```bash
npx @typenode-ai/cli deploy refund-handler.workflow.ts --target=vercel
```

## Architecture in one diagram

```
   Engineer IDE    Chat agent    Visual canvas (read-mostly)
        │              │                │
        └──────────────┼────────────────┘
                       ▼
            ╔══════════════════════╗
            ║  refund.workflow.ts  ║  ← canonical, in your git
            ╚══════════╤═══════════╝
                       │ synthesize()
                       ▼
            ╔══════════════════════╗
            ║   Typenode IR        ║  ← intermediate, typed JSON
            ╚══════════╤═══════════╝
                       │ compile<target>()
                       ▼
   ┌──────────┬──────────┬──────────┬─────────┬──────────────┐
   │    CF    │  Vercel  │  AWS     │ Inngest │  Self-hosted │
   │ Workflow │   WDK    │  SFN     │  fn     │  Node runner │
   └──────────┴──────────┴──────────┴─────────┴──────────────┘
```

Full architecture: [`docs/architecture.md`](./docs/architecture.md). Method surface (v0.1): [`docs/methods-v0.1.md`](./docs/methods-v0.1.md).

## Repository structure

```
packages/
  sdk/                          @typenode-ai/sdk        — the operator surface
  core/                         @typenode-ai/core       — IR + synthesize
  cli/                          @typenode-ai/cli        — typenode synth/deploy/dev
  eslint-plugin/                @typenode-ai/eslint-plugin — determinism rules
  provider-cloudflare/          @typenode-ai/provider-cloudflare
  provider-vercel/              @typenode-ai/provider-vercel
  provider-aws-sfn/             @typenode-ai/provider-aws-sfn
docs/                           Design memos (architecture, methods)
```

## Status

| Surface | Status |
|---|---|
| Architecture spec | ✅ Locked v0.1 |
| Method surface (17 members) | ✅ Locked v0.1 |
| `@typenode-ai/core` synthesize | 🚧 In progress |
| `@typenode-ai/sdk` operators | 🚧 In progress |
| `@typenode-ai/provider-cloudflare` | 🚧 First target |
| `@typenode-ai/provider-vercel` | ⏳ Second target |
| `@typenode-ai/provider-aws-sfn` | ⏳ Third target (portability proof) |
| Documentation site | ⏳ Coming to sdk.typenode.dev |

Real code lands incrementally. Until then, the design memos in [`docs/`](./docs/) are the source of truth — they describe what we're building before it exists.

## Contributing

The SDK is in early design. We're not ready for code contributions yet, but we're very interested in feedback on the [architecture](./docs/architecture.md) and [method surface](./docs/methods-v0.1.md). Open an issue or start a discussion.

See [`CONTRIBUTING.md`](./CONTRIBUTING.md) for details.

## License

Apache-2.0. See [`LICENSE`](./LICENSE).

## About Typenode

Typenode is building cloud-agnostic infrastructure for the agent era. Learn more at [typenode.ai](https://typenode.ai).
