# Typenode SDK & Canonical Workflow Format

**Status:** SEED — May 2026
**Owners:** Bartosz, Michał
**Supersedes:** the current JSONB-graph-as-canonical model (still production today)
**Related:** `projects/typenode-landing/components/deck/slide-*.tsx`, `projects/typenode-api/packages/shared/src/wfp/script-compiler.ts`, `projects/typenode-api/packages/workflow-runner/`

---

## 1. Decision in one line

The canonical source of truth for a Typenode workflow is a **TypeScript file** using `@typenode-ai/sdk`. The TS synthesizes to a typed **intermediate representation (IR)**. Pluggable **compilers** translate the IR to vendor-specific deployment artifacts (Cloudflare Workflows, Vercel WDK, AWS Step Functions, Inngest, self-hosted Node).

This is **Pulumi for agentic workflows**: real programming language as canonical, typed JSON state as intermediate, pluggable providers as backends.

## 2. Why we're here

### The pivot
The deck (`/deck`) positions Typenode as "Cloud-agnostic infrastructure adapter for the agent era." The customer promise: build via chat or code, run on your own cloud, switch in one click. The current JSONB-graph-as-canonical model was designed for a UI-first world (drag-and-drop nodes on a React Flow canvas). It cannot deliver the new promise without an architectural reframe.

### What changes with the agent era
1. **Authoring shifts from canvas to chat + IDE.** LLMs are better at generating TypeScript than at emitting bespoke JSON node mutations.
2. **The customer-owned artifact must be human-readable.** "You own the code" only lands if the code is real, not opaque JSONB.
3. **Engineers and non-engineers need to converge on one substrate.** Two parallel canonicals (visual graph for non-eng, TS for eng) create permanent sync burden. One canonical + multiple views wins.
4. **Multi-cloud requires a portable canonical.** A vendor-flavored canonical (Cloudflare's `step.do`, Inngest's `step.run`) locks you to that vendor's runtime semantics. A typed, vendor-neutral canonical compiles cleanly to each.

### Why not stay with graph JSONB
Visualization is trivial from JSONB, but every other axis loses:
- **Git/diff/review** — JSONB diffs are unreadable
- **Composability** — no modules, no shared libraries
- **LLM authoring** — agents emit graph mutations against a custom schema; less reliable than TS generation
- **Customer asset** — opaque to engineers
- **Multi-vendor portability** — possible but requires double translation (JSONB → TS-shaped IR → vendor)

### Why not vendor-imperative-SDK (Inngest/CF/Vercel style)
- Visualization requires AST analysis of arbitrary code (Cloudflare ships a Rust parser; loops/ternaries/nesting break it)
- Structure is implicit, not extractable — agents can write it but can't safely reason over it
- Vendor lock-in: `step.do` semantics differ per vendor; canonical can't be all of them

### Why TS-with-builder-DSL wins
Strictly dominates both alternatives on every axis except live-edit hot-swap. See §11 for full trade-off accounting.

---

## 3. Architecture

```
   Engineer IDE    Chat agent    Visual canvas (read-mostly)
        │              │                │
        └──────────────┼────────────────┘
                       ▼
            ╔══════════════════════╗
            ║  refund.workflow.ts  ║  ← canonical, in git, in customer's repo
            ║  uses @typenode-ai/sdk  ║
            ╚══════════╤═══════════╝
                       │ synthesize()  (build-time)
                       ▼
            ╔══════════════════════╗
            ║   Typenode IR        ║  ← intermediate, typed JSON
            ║   (compiler input)   ║     stored in DB for tenanted runs
            ╚══════════╤═══════════╝
                       │ compile<target>()  (deploy-time)
                       ▼
   ┌──────────┬──────────┬──────────┬─────────┬──────────────┐
   │    CF    │  Vercel  │  AWS     │ Inngest │  Self-hosted │
   │ Workflow │   WDK    │  SFN     │  fn     │  Node runner │
   └──────────┴──────────┴──────────┴─────────┴──────────────┘
                       │
                       ▼
          Runs on customer's infrastructure
```

### The five surfaces

| Surface | Reads | Writes | Notes |
|---|---|---|---|
| **Engineer IDE** | TS | TS | Customer-owned `.workflow.ts` files, committed to their repo |
| **Chat agent** | TS + IR + run history | TS (as code patches) | The agent never edits JSON directly — it produces TS diffs |
| **Visual canvas** | IR | Lightweight annotations (positions, labels) round-tripped via sidecar | Read-mostly; structural edits regenerate TS |
| **Vendor compilers** | IR | Target artifact | One package per cloud |
| **Runtime** | Target artifact | Run state | Customer-owned; can rip out Typenode and keep running |

---

## 4. The canonical: `@typenode-ai/sdk`

### 4.1 Design principles

1. **Synthesize-time, not runtime.** `s.do(...)`, `s.branch(...)` etc. are *builders*. They record structure into a graph. They never execute the user's `fn` at synth time. The vendor backend executes `fn` at run time.
2. **Statically extractable.** Every control-flow operator (`branch`, `switch`, `parallel`, `loop`, `wait`) is an explicit SDK method call. No native `if`/`for` for workflow structure. Inside `s.do(fn)` closures, native TS is fine — that's leaf code.
3. **Typed end-to-end.** Workflow input/output via Zod. Step outputs propagate as typed `Handle<T>`. Expressions are `Expr<T>`. The IDE catches structural errors at author time.
4. **Web-platform JS for leaves.** Leaf code constrained to `fetch`, standard globals — no `node:fs`, no `node:child_process`. Portable across CF, Vercel Edge, AWS Lambda Node.
5. **Composable.** `s.subWorkflow(...)` lets workflows import workflows. Standard npm modules for shared helpers.

### 4.2 Operator surface

The SDK exposes a small, sharp set of operators. Each maps 1:1 to an IR node type.

```typescript
// @typenode-ai/sdk

import { z, ZodSchema } from "zod";

// ─── Top-level ────────────────────────────────────────────────────────────────

export function workflow<I, O = void>(
  config: {
    id: string;                          // stable identifier (kebab-case)
    input: ZodSchema<I>;
    output?: ZodSchema<O>;
    version?: string;                    // optional, default derived from git
    description?: string;
  },
  build: (s: Scope<I>) => Handle<O> | void,
): WorkflowDef<I, O>;

// ─── Scope: the builder ───────────────────────────────────────────────────────

export interface Scope<Ctx> {
  /** Typed handle to workflow input. Access as `s.input.fieldName`. */
  readonly input: Handle<Ctx>;

  /** A side-effecting operation. Returns a handle to its output. */
  do<Deps extends readonly Handle[], Out>(
    name: string,
    opts: {
      deps?: Deps;                       // explicit dependency declaration
      retries?: RetryPolicy;
      timeout?: Duration;
      mode?: "read" | "write";           // governance signal
      fn: (vals: ResolvedTuple<Deps>) => Promise<Out>;
    },
  ): Handle<Out>;

  /** Two-way conditional branch. */
  branch<T = void, E = void>(opts: {
    when: Expr<boolean>;
    then: (s: Scope<Ctx>) => Handle<T> | void;
    else?: (s: Scope<Ctx>) => Handle<E> | void;
  }): Handle<T | E>;

  /** N-way conditional switch. First matching case wins. */
  switch<K extends string, Out>(opts: {
    cases: Record<K, Expr<boolean>>;
    branches: Record<K, (s: Scope<Ctx>) => Handle<Out> | void>;
    default?: (s: Scope<Ctx>) => Handle<Out> | void;
  }): Handle<Out>;

  /** Concurrent fan-out. All lanes run; result is tuple of handles. */
  parallel<Hs extends readonly (Handle | ((s: Scope<Ctx>) => Handle))[]>(
    ...lanes: Hs
  ): Handle<ResolvedAllTuple<Hs>>;

  /** Iterate over a list. Body produces one output per item. */
  loop<Item, Out>(opts: {
    over: Expr<Item[]>;
    concurrency?: number;                // 1 = sequential; >1 = parallel; default vendor-specific
    body: (s: Scope<Ctx>, item: Handle<Item>) => Handle<Out>;
  }): Handle<Out[]>;

  /** Durable wait — either a fixed duration or an external event. */
  wait(
    opts:
      | { duration: Duration }
      | {
          event: string;                 // event type, max 100 chars
          schema: ZodSchema;
          timeout?: Duration;
          correlation?: Expr<string>;    // filter expression
        },
  ): Handle<unknown>;

  /** Explicit success. Optional value becomes workflow output. */
  succeed<V>(value?: Handle<V> | V): Handle<V | undefined>;

  /** Explicit failure. Terminates workflow with named error. */
  fail(reason: string, opts?: { cause?: Expr<unknown> }): never;

  /** Compose another workflow as a sub-workflow. */
  subWorkflow<I, O>(wf: WorkflowDef<I, O>, input: Handle<I> | Expr<I>): Handle<O>;

  /** Build a typed expression from one or more handles. */
  expr<Args extends readonly Handle[], T>(
    deps: Args,
    compute: (vals: ResolvedTuple<Args>) => T,
  ): Expr<T>;
}

// ─── Types ────────────────────────────────────────────────────────────────────

/** Handle<T> — a synth-time reference to a step's eventual output. */
export interface Handle<T> {
  readonly __brand: "handle";
  readonly __type: T;
  // Property access on Handle returns a sub-handle (Proxy):
  //   const order = s.do("fetch", ...);  // Handle<Order>
  //   order.id                            // Handle<string>
  //   order.items[0].price                // Handle<number>
}

/** Expr<T> — a typed expression composed from handles. */
export interface Expr<T> {
  readonly __brand: "expr";
  readonly __type: T;
}

export type Duration =
  | `${number}s` | `${number}m` | `${number}h` | `${number}d`
  | `${number} seconds` | `${number} minutes` | `${number} hours` | `${number} days`
  | { milliseconds: number };

export interface RetryPolicy {
  max: number;                           // 0 = no retries
  backoff: "constant" | "linear" | "exponential";
  delay: Duration;
  retryOn?: (err: unknown) => boolean;   // optional predicate (compiled to expr)
}
```

### 4.3 Example: refund handler

```typescript
// refund-handler.workflow.ts
import { workflow } from "@typenode-ai/sdk";
import { z } from "zod";

const Order = z.object({
  id: z.string(),
  status: z.enum(["pending", "fulfilled", "cancelled"]),
  chargeId: z.string(),
  totalCents: z.number().int(),
  createdAt: z.string().datetime(),
});

export default workflow({
  id: "refund-handler",
  input: z.object({ orderId: z.string() }),
  output: z.object({ refundId: z.string().nullable(), reason: z.string().nullable() }),
}, (s) => {
  const order = s.do("fetch-order", {
    deps: [s.input.orderId],
    fn: async ([orderId]) => {
      const res = await fetch(`https://api.shop.com/orders/${orderId}`);
      if (!res.ok) throw new Error(`order fetch failed: ${res.status}`);
      return Order.parse(await res.json());
    },
  });

  const canRefund = s.expr([order], ([o]) =>
    o.status === "fulfilled"
    && (Date.now() - new Date(o.createdAt).getTime()) < 30 * 86400_000
  );

  return s.branch({
    when: canRefund,
    then: (s) => {
      const refund = s.do("issue-refund", {
        deps: [order],
        mode: "write",
        retries: { max: 3, backoff: "exponential", delay: "5s" },
        fn: async ([o]) => stripe.refunds.create({ charge: o.chargeId }),
      });
      return s.succeed(s.expr([refund], ([r]) => ({ refundId: r.id, reason: null })));
    },
    else: (s) => s.succeed({ refundId: null, reason: "refund-window-expired" }),
  });
});
```

### 4.4 Example: agent triage with parallel + wait

```typescript
// support-triage.workflow.ts
import { workflow } from "@typenode-ai/sdk";
import { z } from "zod";

export default workflow({
  id: "support-triage",
  input: z.object({ ticketId: z.string(), customerEmail: z.string().email() }),
}, (s) => {
  const ticket = s.do("load-ticket", {
    deps: [s.input.ticketId],
    fn: async ([id]) => loadTicket(id),
  });

  // Parallel: enrich from CRM + classify intent with LLM
  const [crm, intent] = s.parallel(
    (s) => s.do("crm-enrich", {
      deps: [s.input.customerEmail],
      fn: async ([email]) => attio.findContact(email),
    }),
    (s) => s.do("classify-intent", {
      deps: [ticket],
      fn: async ([t]) => llm.classify(t.body, ["billing", "technical", "refund", "other"]),
    }),
  );

  return s.switch({
    cases: {
      refund: s.expr([intent], ([i]) => i === "refund"),
      billing: s.expr([intent], ([i]) => i === "billing"),
      technical: s.expr([intent], ([i]) => i === "technical"),
    },
    branches: {
      refund: (s) => s.subWorkflow(refundHandler, s.expr([ticket], ([t]) => ({ orderId: t.orderId }))),
      billing: (s) => s.do("notify-billing", {
        deps: [ticket, crm],
        fn: async ([t, c]) => slack.post("#billing", { ticket: t, account: c }),
      }),
      technical: (s) => {
        const approval = s.wait({
          event: "engineer-acknowledged",
          schema: z.object({ engineerId: z.string() }),
          timeout: "4h",
          correlation: s.expr([ticket], ([t]) => t.id),
        });
        return s.do("assign-engineer", {
          deps: [ticket, approval],
          fn: async ([t, a]) => assignToEngineer(t.id, (a as any).engineerId),
        });
      },
    },
    default: (s) => s.do("auto-reply", {
      deps: [ticket],
      fn: async ([t]) => autoReply(t),
    }),
  });
});
```

This is **statically analyzable**: every operator call is an SDK method, every reference goes through `Handle` or `Expr`. The graph can be extracted without parsing the user's `fn` bodies.

---

## 5. The intermediate representation (IR)

### 5.1 Schema (TypeScript, source of truth)

```typescript
// @typenode-ai/core/ir.ts

export interface WorkflowIR {
  schemaVersion: "1";                    // bump on breaking IR changes
  id: string;
  version: string;
  input: JsonSchema;
  output: JsonSchema | null;
  nodes: IRNode[];
  // Sidecar (optional): canvas positions, comments — does not affect execution
  sidecar?: {
    positions?: Record<string, { x: number; y: number }>;
    labels?: Record<string, string>;
  };
}

export type IRNode =
  | IRDo | IRBranch | IRSwitch | IRParallel | IRLoop
  | IRWaitDuration | IRWaitEvent | IRSucceed | IRFail | IRSubWorkflow;

interface IRNodeBase {
  id: string;                            // unique within workflow
  description?: string;
  depends_on: NodeRef[];                 // explicit data deps (also derives edge graph)
}

export interface IRDo extends IRNodeBase {
  op: "do";
  mode: "read" | "write";
  code: CodeRef;                         // ref to a code blob (or inline if small)
  retries?: RetryPolicy;
  timeout_ms?: number;
}

export interface IRBranch extends IRNodeBase {
  op: "branch";
  when: ExprRef;
  then_first: string;                    // id of first node in then-branch (null = no-op)
  else_first: string | null;
}

export interface IRSwitch extends IRNodeBase {
  op: "switch";
  cases: Array<{ key: string; when: ExprRef; first: string }>;
  default_first: string | null;
}

export interface IRParallel extends IRNodeBase {
  op: "parallel";
  lanes: string[];                       // ids of first node in each lane
  join_id: string;                       // synthetic merge node id
}

export interface IRLoop extends IRNodeBase {
  op: "loop";
  over: ExprRef;
  body_first: string;
  concurrency: number;                   // 1 = sequential
  item_alias: string;                    // name the body can reference
}

export interface IRWaitDuration extends IRNodeBase {
  op: "wait_duration";
  duration_ms: number;
}

export interface IRWaitEvent extends IRNodeBase {
  op: "wait_event";
  event_type: string;
  schema: JsonSchema;
  timeout_ms: number | null;
  correlation?: ExprRef;
}

export interface IRSucceed extends IRNodeBase {
  op: "succeed";
  value?: ExprRef;                       // optional output value
}

export interface IRFail extends IRNodeBase {
  op: "fail";
  reason: string;
  cause?: ExprRef;
}

export interface IRSubWorkflow extends IRNodeBase {
  op: "sub_workflow";
  workflow_ref: string;                  // versioned ref
  input: ExprRef;
}

// References ──────────────────────────────────────────────────────────────────

export interface NodeRef {
  node_id: string;
  path?: string[];                       // sub-path into output: ["items", "0", "price"]
}

export interface CodeRef {
  /** Inline only for tiny bodies (<1KB). Otherwise blob:// pointer. */
  inline?: string;
  blob?: string;                         // content-addressed (sha256)
  language: "javascript-web";            // constrained to Web-platform JS
}

export type ExprRef =
  | { kind: "node_ref"; ref: NodeRef }
  | { kind: "input_ref"; path: string[] }
  | { kind: "literal"; value: JsonValue }
  | { kind: "jsonata"; expr: string; deps: NodeRef[] }
  | { kind: "js_pure"; body: CodeRef; deps: NodeRef[] };

export interface RetryPolicy {
  max: number;
  backoff: "constant" | "linear" | "exponential";
  delay_ms: number;
  retry_on?: ExprRef;
}
```

### 5.2 Example: refund handler IR

The refund example in §4.3 synthesizes to (abbreviated):

```json
{
  "schemaVersion": "1",
  "id": "refund-handler",
  "version": "1.0.0",
  "input": { "type": "object", "properties": { "orderId": { "type": "string" } }, "required": ["orderId"] },
  "output": { "type": "object", "properties": { "refundId": { "type": ["string", "null"] }, "reason": { "type": ["string", "null"] } } },
  "nodes": [
    {
      "id": "fetch-order",
      "op": "do",
      "mode": "read",
      "depends_on": [{ "node_id": "@input", "path": ["orderId"] }],
      "code": { "blob": "sha256:abc...", "language": "javascript-web" }
    },
    {
      "id": "branch_0",
      "op": "branch",
      "depends_on": [{ "node_id": "fetch-order" }],
      "when": {
        "kind": "js_pure",
        "body": { "inline": "([o]) => o.status === 'fulfilled' && (Date.now() - new Date(o.createdAt).getTime()) < 30 * 86400000", "language": "javascript-web" },
        "deps": [{ "node_id": "fetch-order" }]
      },
      "then_first": "issue-refund",
      "else_first": "succeed_else_0"
    },
    {
      "id": "issue-refund",
      "op": "do",
      "mode": "write",
      "depends_on": [{ "node_id": "fetch-order" }],
      "retries": { "max": 3, "backoff": "exponential", "delay_ms": 5000 },
      "code": { "blob": "sha256:def...", "language": "javascript-web" }
    },
    {
      "id": "succeed_then_0",
      "op": "succeed",
      "depends_on": [{ "node_id": "issue-refund" }],
      "value": {
        "kind": "js_pure",
        "body": { "inline": "([r]) => ({ refundId: r.id, reason: null })", "language": "javascript-web" },
        "deps": [{ "node_id": "issue-refund" }]
      }
    },
    {
      "id": "succeed_else_0",
      "op": "succeed",
      "depends_on": [{ "node_id": "branch_0" }],
      "value": { "kind": "literal", "value": { "refundId": null, "reason": "refund-window-expired" } }
    }
  ]
}
```

Two important properties:
- **All structure is explicit.** No AST analysis needed to draw the graph or compile to a target.
- **Code blobs are content-addressed.** A workflow that uses the same `fn` body twice stores it once. Important when leaves are shipped to multiple vendor targets.

### 5.3 Versioning

- `schemaVersion` bumped on breaking IR changes; compilers must declare which versions they accept.
- Workflow `version` bumped on every deploy. Old running instances reference their snapshot of the IR.
- IR is **internal**. Public-facing customers see TS + the deployed artifact. The IR exists so compilers have a stable target.

---

## 6. The synthesize model

This is the most subtle piece. Anyone working on the SDK needs to internalize it.

### 6.1 Lifecycle

```
1. User runs `typenode synth refund-handler.workflow.ts`  (CLI)
   or `import wf from "./refund-handler.workflow.ts"`     (programmatic)

2. The TS module is loaded. The default export is a `WorkflowDef`.

3. `WorkflowDef` is the return value of `workflow(config, build)`.

4. `workflow(...)` invokes `build(s)` ONCE, with a recording Scope.

5. Every `s.do(...)`, `s.branch(...)`, etc. call appends a node to the graph
   and returns a Handle/Expr that records the reference path.

6. The recording Scope freezes the graph and emits IR.

7. The leaf `fn` closures are NOT executed. They are extracted as source
   (via Function.prototype.toString or a build-time TS transform — see §6.4)
   and stored as code blobs in the IR.
```

### 6.2 Handles as proxies

`Handle<T>` is implemented as a `Proxy`. Property access on a handle returns a new sub-handle that records the access path:

```typescript
const order = s.do("fetch-order", ...);   // Handle<Order>
order.id                                  // Handle<string>, path = ["id"]
order.items[0].price                      // Handle<number>, path = ["items", 0, "price"]
```

When a handle is passed to another operator (`deps:`, `expr(...)`, `succeed(...)`), the SDK extracts the recorded path and stores it as a `NodeRef`.

This is the trick that makes the SDK feel like real code without actually executing anything.

### 6.3 Expressions

`s.expr(deps, compute)` is the typed expression DSL. Compilation differs by target:

- **CF / Vercel / Inngest / self-hosted**: `compute` is shipped verbatim as a `js_pure` expression. Runs against resolved values at runtime.
- **AWS Step Functions**: `compute` is transpiled to JSONata where possible (simple member access, comparisons, arithmetic). Complex expressions fall back to a tiny "expression Lambda."

The build-time check ensures `compute` references only its declared `deps` (no closure capture of mutable variables). Closures over module-level constants are fine.

### 6.4 Closure extraction

Two strategies, both well-trodden:

**Strategy A — runtime extraction via `Function.prototype.toString()`.**
Works for simple cases. Fails when bundlers minify or when the function references outer-scope bindings.

**Strategy B — build-time TS plugin.**
A `ts-patch` / SWC plugin that walks the AST, detects `s.do(...)` / `s.expr(...)` callbacks, lifts them to top-level functions, and rewrites the source to reference the lifted symbols. Cleaner, more robust, no minification issues.

**Recommendation: ship Strategy B from day one.** Pulumi/CDK both went through the "let's try `toString()` first" phase and ended up with build-time transforms. Skip the detour.

### 6.5 Determinism guarantees

The build function must be deterministic — same source, same IR. Enforced by:
- No `Date.now()`, `Math.random()`, env vars in the build function (lint rule)
- Stable node IDs from user-provided names; synthetic names use deterministic counters
- IR serialization is canonical (sorted keys, normalized whitespace)

---

## 7. Compilers

### 7.1 Compiler interface

```typescript
// @typenode-ai/core/compiler.ts

export interface Compiler<TargetArtifact> {
  readonly id: string;                   // "cloudflare-workflows" | "vercel-wdk" | "aws-sfn" | ...
  readonly schemaVersionsSupported: string[];

  compile(ir: WorkflowIR, opts: CompileOptions): TargetArtifact;
  capabilities(): CompilerCapabilities;
}

export interface CompilerCapabilities {
  maxStepDurationMs: number;
  maxWaitDurationMs: number;
  supportsParallelism: boolean;
  supportsLoops: boolean;
  supportsSubWorkflows: boolean;
  supportsEventWait: boolean;
  // ... per-vendor limits
}
```

The capabilities API lets the UI warn users at synth time when a workflow uses a feature its target doesn't support (e.g., "AWS Step Functions Express has a 5-minute max; your loop body could exceed that").

### 7.2 Built-in compilers (roadmap)

| Package | Target | Status | First worked example |
|---|---|---|---|
| `@typenode-ai/provider-cloudflare` | CF Workflows + WFP dispatch Worker | **First ship** — reuse `wfp/script-compiler.ts` logic | §7.3 |
| `@typenode-ai/provider-vercel` | Vercel WDK (`"use workflow"` / `"use step"` TS) | Second ship | §7.4 |
| `@typenode-ai/provider-aws-sfn` | AWS Step Functions ASL JSON + per-leaf Lambdas | Third ship — strongest portability proof | §7.5 |
| `@typenode-ai/provider-inngest` | Inngest function w/ `step.run` | Later |  |
| `@typenode-ai/provider-self-hosted` | Standalone Node runner | Later — needed for OSS / "rip out Typenode" story |  |

### 7.3 Worked example: IR → Cloudflare Workflows TS

Input: refund-handler IR (§5.2).

Output (compiler emits this file as part of the deploy bundle):

```typescript
// generated by @typenode-ai/provider-cloudflare — DO NOT EDIT
import { WorkflowEntrypoint, type WorkflowEvent, type WorkflowStep } from "cloudflare:workers";
import { dispatch } from "./_wfp-dispatch.js";

interface Input { orderId: string }

export class RefundHandlerWorkflow extends WorkflowEntrypoint<Env, Input> {
  async run(event: WorkflowEvent<Input>, step: WorkflowStep) {
    const order = await step.do("fetch-order", async () =>
      dispatch(this.env, "refund-handler", "fetch-order", { orderId: event.payload.orderId })
    );

    const canRefund =
      order.status === "fulfilled"
      && (Date.now() - new Date(order.createdAt).getTime()) < 30 * 86400_000;

    if (canRefund) {
      const refund = await step.do("issue-refund", {
        retries: { limit: 3, backoff: "exponential", delay: "5 seconds" },
      }, async () => dispatch(this.env, "refund-handler", "issue-refund", { order }));
      return { refundId: refund.id, reason: null };
    }
    return { refundId: null, reason: "refund-window-expired" };
  }
}
```

Leaf code (`fetch-order`, `issue-refund`) is bundled into a per-workflow WFP Worker (see `wfp/script-compiler.ts` for the existing pattern). The CF Workflow class above is the orchestrator; it calls leaves via service binding.

### 7.4 Worked example: IR → Vercel WDK TS

```typescript
// generated by @typenode-ai/provider-vercel — DO NOT EDIT
import { z } from "zod";

const fetchOrder = async (orderId: string) => {
  "use step";
  const res = await fetch(`https://api.shop.com/orders/${orderId}`);
  if (!res.ok) throw new Error(`order fetch failed: ${res.status}`);
  return await res.json();
};

const issueRefund = async (order: { chargeId: string }) => {
  "use step";
  return stripe.refunds.create({ charge: order.chargeId });
};
issueRefund.maxRetries = 3;

export async function refundHandlerWorkflow(input: { orderId: string }) {
  "use workflow";
  const order = await fetchOrder(input.orderId);
  const canRefund =
    order.status === "fulfilled"
    && (Date.now() - new Date(order.createdAt).getTime()) < 30 * 86400_000;
  if (canRefund) {
    const refund = await issueRefund(order);
    return { refundId: refund.id, reason: null };
  }
  return { refundId: null, reason: "refund-window-expired" };
}
```

### 7.5 Worked example: IR → AWS Step Functions ASL

```json
{
  "Comment": "refund-handler v1.0.0",
  "StartAt": "fetch-order",
  "States": {
    "fetch-order": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": { "FunctionName": "${RefundHandler_FetchOrder_Lambda}", "Payload.$": "$" },
      "ResultPath": "$.order",
      "Next": "branch_0",
      "Retry": [{ "ErrorEquals": ["States.ALL"], "MaxAttempts": 3, "BackoffRate": 2, "IntervalSeconds": 1 }]
    },
    "branch_0": {
      "Type": "Choice",
      "Choices": [{
        "And": [
          { "Variable": "$.order.Payload.status", "StringEquals": "fulfilled" },
          { "Variable": "$.order.Payload.createdAtMs", "NumericLessThan": 2592000000 }
        ],
        "Next": "issue-refund"
      }],
      "Default": "succeed_else_0"
    },
    "issue-refund": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": { "FunctionName": "${RefundHandler_IssueRefund_Lambda}", "Payload.$": "$" },
      "ResultPath": "$.refund",
      "Retry": [{ "ErrorEquals": ["States.ALL"], "MaxAttempts": 3, "BackoffRate": 2, "IntervalSeconds": 5 }],
      "Next": "succeed_then_0"
    },
    "succeed_then_0": {
      "Type": "Pass",
      "Parameters": { "refundId.$": "$.refund.Payload.id", "reason": null },
      "End": true
    },
    "succeed_else_0": {
      "Type": "Pass",
      "Result": { "refundId": null, "reason": "refund-window-expired" },
      "End": true
    }
  }
}
```

(Leaf Lambdas deployed separately; expression simplification — `$.order.Payload.createdAtMs < 30d` — emitted by the compiler from the `js_pure` expr.)

**When the AWS compiler can emit clean ASL from your IR without ad-hoc IR extensions, your portability story is real.** This is the truest test.

---

## 8. The leaf JS contract

To keep `fn` bodies portable across CF Workers, Vercel Edge/Node, AWS Lambda Node, and self-hosted Node:

**Allowed**
- All ES2022+ syntax
- `fetch`, `Response`, `Request`, `Headers`, `URL`, `URLSearchParams`
- `crypto.subtle`, `crypto.randomUUID()`
- `TextEncoder`, `TextDecoder`, `atob`, `btoa`
- `console.*` (captured by runner)
- `JSON.parse`/`stringify`
- Standard collections (`Map`, `Set`, `Array`, etc.)
- Async/await, generators

**Forbidden**
- `node:*` built-ins (`fs`, `child_process`, `dgram`, `net`, ...)
- `eval`, `Function()` constructor (CF Workers block these anyway)
- Native bindings (anything `node-gyp`-compiled)
- Process-level globals (`process.env` — use injected `secrets` parameter instead)
- Filesystem assumptions

**Injected per node**
- `secrets` — pre-decrypted secrets bag, scoped to the leaf (existing pattern, see `wfp/script-compiler.ts`)
- `console` — capturing console (100 entries / 256KB cap; existing pattern)
- Dependency values — resolved values for the declared `deps`

This matches what `wfp/script-compiler.ts` already enforces. The contract is documented as part of `@typenode-ai/sdk`, validated by an ESLint plugin (`eslint-plugin-typenode`).

---

## 9. Migration from current JSONB graph

A one-way decompiler from the current JSONB schema to `@typenode-ai/sdk` TS. The node types map directly:

| Current `NodeType` | Decompiled SDK call |
|---|---|
| `input` | `workflow({ input: zod-from-input-schema }, ...)` |
| `output` | `s.succeed(...)` |
| `action` | `s.do(name, { mode: "write", fn })` |
| `transformation` | `s.do(name, { mode: "read", fn })` |
| `decision` (N handles) | `s.switch(...)` (N branches) |
| `merge` | implicit (next sequential op) |
| `wait_for_input` | `s.wait({ event, timeout })` |
| `loop` | `s.loop({ over, body })` |
| `terminal` (legacy) | `s.fail(...)` or `s.succeed()` |

The decompiler walks the JSONB graph in topological order, emits TS, runs Prettier, ships the result as `.workflow.ts` in the customer's workspace. Customers see a "Migrate to SDK" button. Existing JSONB workflows continue to run on the legacy path until migrated; the legacy runner retires once <5% of active workflows remain on JSONB.

---

## 10. CLI

```bash
typenode init                          # Scaffold project (package.json, tsconfig, sample workflow)
typenode synth <file>.workflow.ts      # TS → IR (writes to .typenode/ir/<id>.json)
typenode synth --all                   # Synth every workflow in src/
typenode diff <id>                     # Diff committed IR vs current synthesis
typenode validate <file>               # Type-check + structural validation, no synth output
typenode dev                           # Local dev server: synth + run + canvas viewer at localhost
typenode deploy --target=cloudflare    # Synth + compile + deploy
typenode deploy --target=vercel
typenode deploy --target=aws-sfn
typenode runs list <id>                # List recent runs of a workflow
typenode runs inspect <run-id>         # Show run timeline + per-step IO
```

Modeled after Pulumi (`pulumi up`) and Wrangler (`wrangler deploy`).

---

## 11. Trade-offs being accepted

| Trade-off | Why we accept |
|---|---|
| Visual canvas demoted to read-mostly | Chat + IDE eat visual editing in agent era; demo flow stays canvas-first |
| Live-edit (CMS-style node tweaks) requires re-synth | Acceptable for engineers; for non-engineers, "parameters" sub-document exposed as live config |
| SDK design is load-bearing — get it wrong and everything suffers | Heavy prior art (CDK, Pulumi, Mastra, Eventual); design space well-trodden |
| Adding new step types is an SDK release | Better than today's silent schema bumps; semver discipline forces clarity |
| One-way migration from JSONB | Mechanical decompile is feasible; customers see a button, not a chore |
| At least one breaking SDK release in year one | Pulumi/CDK both had this; communicated upfront |

## 12. Anti-patterns to refuse

1. **Don't make `s.do(...)` return a real `Promise<T>` you can `await` at synth time.** It returns a `Handle<T>`. Awaiting it is a runtime-only concept on the compiled output, not in the canonical.
2. **Don't allow native `if`/`for` for workflow structure.** Inside `s.do(fn)` closures: fine. Between operators: use `s.branch`/`s.switch`/`s.loop`.
3. **Don't expose IR as a public authoring surface.** It's a compiler intermediate. If customers start hand-editing IR JSON, you have two canonicals.
4. **Don't ship multiple SDK shapes** (imperative for engineers, fluent for low-code). One SDK. The fluent operators are the engineer experience too.
5. **Don't ship vendor backends before the SDK is locked.** Order: SDK spec → IR → CF compiler (existing) → Vercel compiler (second cloud) → AWS SFN compiler (portability proof).
6. **Don't conflate `wait_duration` and `wait_event` in IR.** Different vendor primitives, different cost semantics.
7. **Don't allow closure capture of mutable outer-scope variables in `fn`.** Lint at author time. Module-level constants and explicit `deps:` only.

---

## 13. Open questions

These need resolution before implementation, ordered by blocking-ness:

1. **Closure extraction strategy.** Strategy A (`toString()`) vs Strategy B (build-time TS plugin). Recommendation: B. Decision needed.
2. **Expression language for AWS targets.** JSONata vs JSONPath vs always-fall-back-to-Lambda. JSONata is more expressive; Lambda fallback is universal but slower/costlier. Likely answer: JSONata for simple expressions, Lambda for complex.
3. **How does `secrets` work cross-vendor?** Today: D1 credential vault → injected per node by WFP dispatch. Vercel: env vars at deploy. AWS: Secrets Manager. Need a unified `Secret` reference in IR that each compiler translates.
4. **Sub-workflow versioning.** Does `s.subWorkflow(refundHandler, ...)` pin to a specific version of `refundHandler`, or auto-update? Pulumi pins via lockfile; we should too.
5. **The visual canvas mutation model.** When a user repositions a node on canvas, where does that get stored? Sidecar JSON co-located with the TS file? Comments in the TS? In the DB only? Recommendation: sidecar `.workflow.layout.json` co-committed with TS.
6. **Run-state representation.** This spec covers definition. Runtime state (executions, step outputs, run history) is unaffected today but should converge to a typed shape too. Out of scope here, separate spec.
7. **Streaming and incremental output.** Vercel WDK has `getWritable()` for streamed output to clients. Should the SDK expose `s.stream(...)`? Defer until customer demand.
8. **Cold-start cost of synth on Cloudflare.** Synth runs in a build step, not at request time — but for the chat agent's "preview" path, we need fast synth. Benchmark target: <200ms synth for typical workflows.
9. **What's in `@typenode-ai/sdk` v0.1.0 vs deferred?** Suggested: ship `do`, `branch`, `parallel`, `wait_duration`, `succeed`, `fail`, `subWorkflow`. Defer `switch`, `loop`, `wait_event` to v0.2 if needed for timeline pressure.
10. **Naming.** Is `s.do` right, or `s.task`? `s.branch` vs `s.if`? Lock conventions before public release; ergonomic differences matter for adoption.

---

## 14. Concrete first-90-days plan

| Week | Deliverable |
|---|---|
| 1–2 | Lock SDK spec at v0.1. Concrete type signatures published. One example workflow per operator in `examples/`. |
| 3–4 | `@typenode-ai/core`: synthesize implementation + IR schema validation + closure extraction (Strategy B). |
| 5–6 | `@typenode-ai/provider-cloudflare`: IR → CF Workflow TS + WFP dispatch. Reuse `wfp/script-compiler.ts`. End-to-end test: same workflow runs identically vs current JSONB path. |
| 7 | Decompiler: current JSONB → `.workflow.ts`. Migrate one production workflow as proof. |
| 8–10 | `@typenode-ai/provider-vercel`: IR → Vercel WDK TS. Second cloud working. |
| 11–12 | Visual canvas re-pointed: `synth(ts) → IR → render`. Read + light edit (rename, reposition) round-tripping via sidecar JSON. |
| Stretch 13+ | `@typenode-ai/provider-aws-sfn`: portability proof. `typenode dev` local runner. |

---

## 15. The deck-level claim, made true

Slide 4 today: *"Build with chat. View the flowchart. Run code in your own cloud."*

Under this architecture, each line becomes a literal description:

- **"Build with chat"** — agent emits TS files; engineer reviews PRs; both work on the same artifact
- **"View the flowchart"** — canvas renders from derived IR; lossless, always in sync with the TS
- **"Run code in your own cloud"** — vendor compiler emits the customer's deployment artifact, which they own and can run independently

The deck headline "Cloud-agnostic infrastructure adapter for the agent era" stops being positioning and becomes load-bearing technical truth: the SDK is the adapter language, the IR is the adapter contract, the compilers are how it adapts.

---

## 16. References

### Internal
- Current canonical types: `projects/typenode-api/packages/shared/src/types/workflow.ts`
- Current vendor compiler (CF only): `projects/typenode-api/packages/shared/src/wfp/script-compiler.ts`
- Current runner: `projects/typenode-api/packages/workflow-runner/src/workflows/runner/`
- Pivot deck: `projects/typenode-landing/components/deck/slide-*.tsx`

### External (research synthesized into this spec)
- AWS CDK Step Functions: `aws-cdk-lib/aws-stepfunctions` — the most direct prior art for "TS that synthesizes to canonical JSON"
- Pulumi: real-language IaC with typed state — the conceptual model this spec follows
- Cloudflare Workflows: `step.do` API, replay model, AST-based visualization (the cautionary tale)
- Vercel WDK: `"use workflow"` / `"use step"` directives
- AWS Step Functions ASL: declarative JSON IR — the portability proof we'll target third
- Inngest, Trigger.dev: imperative-with-memoization SDKs (compilation targets, not canonical patterns)
- Mastra: fluent TS workflow API — closest contemporary to our SDK shape
- n8n: graph JSON canonical (today's Typenode) — what we're moving away from
