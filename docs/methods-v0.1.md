# Typenode SDK v0.1 — Locked Method Surface

**Status:** LOCKED for v0.1
**Decided:** 2026-05-18
**Owners:** Bartosz, Michał
**Companion doc:** `typenode-sdk-canonical.md` (broader architecture)

---

## Decision

The `@typenode-ai/sdk` v0.1 ships **17 members on the `step` object**:
- 9 operators (produce IR nodes)
- 3 handles / helpers (produce references and expressions)
- 5 determinism + observability primitives

Anything outside this list is **out of scope for v0.1**. See §6.

---

## 1. The locked surface

| Category | Member | Type | Produces |
|---|---|---|---|
| **Operators (9)** | `step.do` | method | IR node (`do`) |
| | `step.route` | method | IR node (`route`) |
| | `step.all` | method | IR node (`all`) |
| | `step.loop` | method | IR node (`loop`) |
| | `step.pause` | method | IR node (`pause`) |
| | `step.listen` | method | IR node (`listen`) |
| | `step.stop` | method | IR node (`stop`) |
| | `step.output` | method | IR node (`output`) |
| | `step.wf` | method | IR node (`wf`) |
| **Helpers (3)** | `step.input` | property | `Handle<I>` |
| | `step.secret` | method | secret reference |
| | `step.expr` | method | `Expr<T>` |
| **Determinism + Observability (5)** | `step.log` | method | run-timeline entry |
| | `step.now` | method | deterministic timestamp |
| | `step.random` | method | deterministic random number |
| | `step.uuid` | method | deterministic UUID |
| | `step.meta` | property | run metadata bag |

---

## 2. Per-method spec

### 2.1 Operators

#### `step.do(name, opts) → Handle<T>`

```typescript
step.do<Deps extends readonly Handle[], Out>(
  name: string,                            // unique within the workflow scope
  opts: {
    deps?: Deps;
    fn: (vals: ResolvedTuple<Deps>) => Promise<Out>;
    retries?: RetryPolicy;                 // default: { max: 3, backoff: "exponential", delay: "1s" }
    timeout?: Duration;                    // default: vendor-specific (CF: 30 min)
    mode?: "read" | "write";               // default: "read"; governance signal
  },
): Handle<Out>;
```

- **Canvas label:** `Do: <name>` (e.g., `Do: fetch-order`)
- **Compiles to:** CF `step.do`; Vercel `"use step"` fn; ASL `Task` state; Inngest `step.run`

#### `step.route({ cases, default }) → Handle`

```typescript
step.route<Out>(opts: {
  cases: Array<{
    name: string;                          // case identifier
    when: Expr<boolean>;
    then: (s: Scope) => Handle<Out> | void;
  }>;
  default: (s: Scope) => Handle<Out> | void;   // required
}): Handle<Out>;
```

- Cases evaluated **in order, first match wins**.
- **Canvas label:** `Route: <case-names joined by |>` (e.g., `Route: enterprise | midmarket`)
- **Compiles to:** CF/Vercel/Inngest `if/else if/else`; ASL `Choice` state

#### `step.all(...lanes) → Handle<tuple>`

```typescript
step.all<Lanes extends readonly ((s: Scope) => Handle)[]>(
  ...lanes: Lanes
): Handle<ResolvedAllTuple<Lanes>>;
```

- All lanes execute concurrently; resolves when **all** complete.
- **Canvas label:** `All · <N> lanes` (e.g., `All · 3 lanes`) — see §5 mitigations
- **Compiles to:** CF/Vercel/Inngest `Promise.all([...])`; ASL `Parallel` state

#### `step.loop({ over, body, concurrency? }) → Handle<Out[]>`

```typescript
step.loop<Item, Out>(opts: {
  over: Expr<Item[]>;
  body: (s: Scope, item: Handle<Item>) => Handle<Out>;
  concurrency?: number;                    // default: 1 (sequential)
}): Handle<Out[]>;
```

- **Canvas label:** `Loop · <length-expr-if-known>` (e.g., `Loop · 10 items`)
- **Compiles to:** CF/Vercel/Inngest `for ... of` + step calls; ASL `Map` state

#### `step.pause({ duration }) → void`

```typescript
step.pause(opts: { duration: Duration }): void;
```

- Durable sleep — vendor backend does not consume compute during pause.
- **Canvas label:** `Pause · <duration>` (e.g., `Pause · 7d`)
- **Compiles to:** CF `step.sleep`; Vercel `sleep`; ASL `Wait` state; Inngest `step.sleep`
- **Max duration:** vendor-clamped (CF: 365 days; AWS: 1 year)

#### `step.listen({ event, schema, timeout?, correlation? }) → Handle<T>`

```typescript
step.listen<T>(opts: {
  event: string;                           // event type name, max 100 chars, [a-z0-9_-]
  schema: ZodSchema<T>;
  timeout?: Duration;                      // default: "24h"
  correlation?: Expr<string>;              // filter expression
}): Handle<T>;
```

- Suspends until matching event arrives; vendor-backend does not consume compute during wait.
- **Canvas label:** `Listen · <event-name>` (e.g., `Listen · reviewer-response`)
- **Compiles to:** CF `step.waitForEvent`; Vercel hook + `for await`; ASL `Wait` + task-token callback; Inngest `step.waitForEvent`

#### `step.stop(reason, opts?) → never`

```typescript
step.stop(
  reason: string,                          // error code, kebab-case
  opts?: { cause?: Expr<unknown> },
): never;
```

- Terminates workflow with named error.
- **Canvas label:** `Stop · <reason>` (e.g., `Stop · refund-window-expired`)
- **Compiles to:** CF/Vercel/Inngest `throw new TypenodeStopError(reason)`; ASL `Fail` state

#### `step.output(value?) → void`

```typescript
step.output<V>(value?: Handle<V> | V): void;
```

- Terminates workflow with success; value becomes workflow output.
- **Canvas label:** `Output` (or `Output: <key-list>` if value is object)
- **Compiles to:** CF/Vercel/Inngest `return value`; ASL `Pass` + `End: true`

#### `step.wf(otherWorkflow, input) → Handle<O>`

```typescript
step.wf<I, O>(
  otherWorkflow: WorkflowDef<I, O>,
  input: Handle<I> | Expr<I>,
): Handle<O>;
```

- Composes another workflow as a sub-step. Sub-workflow gets its own run record.
- **Canvas label:** `Workflow: <other-workflow-id>` — see §5 mitigations
- **Compiles to:** CF/Vercel sub-Workflow invocation; ASL `Task` with `StartExecution.sync2`; Inngest `step.invoke`

### 2.2 Helpers

#### `step.input → Handle<I>`

- Typed handle to the workflow's input.
- Access via property chains: `step.input.orderId`, `step.input.customer.email`.
- Compiles to runtime `event.payload.<path>` (CF/Vercel) or `$.input.<path>` (ASL).

#### `step.secret(name) → SecretRef`

```typescript
step.secret(name: string): SecretRef;
```

- Cross-vendor secret reference. Passes through to leaf `fn` as a string value.
- **Compiles to:** CF D1 vault dispatch; Vercel env var; AWS Secrets Manager fetch; Inngest env var
- **Names:** uppercase snake_case (`STRIPE_API_KEY`, `OPENAI_KEY`)

#### `step.expr(deps, compute) → Expr<T>`

```typescript
step.expr<Deps extends readonly Handle[], T>(
  deps: Deps,
  compute: (vals: ResolvedTuple<Deps>) => T,
): Expr<T>;
```

- Builds a typed expression from one or more handles.
- The `compute` closure runs **at runtime against resolved values**, NOT at synth time.
- Closure must reference only its declared `deps` (no outer-scope variable capture). Enforced by ESLint rule.
- **Compiles to:** CF/Vercel/Inngest inline JS; ASL JSONata where expressible, evaluator-Lambda fallback otherwise.

### 2.3 Determinism + Observability

#### `step.log(level, message, fields?) → void`

```typescript
step.log(
  level: "debug" | "info" | "warn" | "error",
  message: string,
  fields?: Record<string, unknown>,
): void;
```

- Structured log entry. Captured to the run timeline (not the leaf's `console`).
- **Visible in:** Typenode dashboard run viewer + vendor-specific log target.

#### `step.now() → Handle<number>`

```typescript
step.now(): Handle<number>;                // ms since epoch
```

- Deterministic current time. Captured once on first execution; replayed identically on retry.
- **DO NOT use `Date.now()` inside `step.do(fn)` for any value that affects control flow.** ESLint rule enforces this.

#### `step.random() → Handle<number>`

```typescript
step.random(): Handle<number>;             // [0, 1)
```

- Deterministic random. Captured once, replayed identically.

#### `step.uuid() → Handle<string>`

```typescript
step.uuid(): Handle<string>;               // RFC 4122 v4
```

- Deterministic UUID. Captured once, replayed identically.

#### `step.meta → RunMeta`

```typescript
step.meta: {
  runId: string;
  workflowId: string;
  workflowVersion: string;
  tenantId: string;
  attempt: number;                         // 0-indexed retry attempt
  traceId?: string;
  deployedAt: string;                      // ISO 8601
};
```

- Read-only run metadata. Useful for correlation IDs in logs / outbound requests.

---

## 3. Type machinery (locked)

```typescript
export interface Handle<T> {
  readonly __brand: "handle";
  readonly __type: T;
  // Proxy: property access returns sub-handle (path recorded)
}

export interface Expr<T> {
  readonly __brand: "expr";
  readonly __type: T;
}

export interface SecretRef {
  readonly __brand: "secret";
  readonly name: string;
}

export type Duration =
  | `${number}s` | `${number}m` | `${number}h` | `${number}d`
  | `${number} seconds` | `${number} minutes` | `${number} hours` | `${number} days`
  | { milliseconds: number };

export interface RetryPolicy {
  max: number;                             // 0 disables retry
  backoff: "constant" | "linear" | "exponential";
  delay: Duration;
  retryOn?: (err: unknown) => boolean;     // compiled to expr; restricted to deterministic predicates
}

type ResolvedTuple<Deps extends readonly Handle[]> = {
  [K in keyof Deps]: Deps[K] extends Handle<infer T> ? T : never;
};

type ResolvedAllTuple<Lanes extends readonly ((s: any) => Handle)[]> = {
  [K in keyof Lanes]: Lanes[K] extends (s: any) => Handle<infer T> ? T : never;
};
```

---

## 4. Defaults reference

| Operator | Default | Why |
|---|---|---|
| `step.do` retries | `{ max: 3, backoff: "exponential", delay: "1s" }` | Standard transient-failure handling |
| `step.do` timeout | Vendor max (CF: 30min) | Don't pre-truncate vendor capabilities |
| `step.do` mode | `"read"` | Safer default; "write" must be explicit |
| `step.listen` timeout | `"24h"` | Most human-in-the-loop responds within a day |
| `step.loop` concurrency | `1` (sequential) | Deterministic ordering by default |
| `step.route` | (no default fallthrough — `default` is required) | Forces exhaustive thinking |

---

## 5. Canvas rendering mitigations

Two operator names are short enough that they require canvas-layer translation to remain operator-friendly:

| Method | Naked label | Canvas renders as | Tooltip / detail panel |
|---|---|---|---|
| `step.all` | "All" | `All · 3 lanes` (count appended) | "Runs concurrently. Waits for all to complete." |
| `step.wf` | "WF" | `Workflow: <id>` (full word + name) | Shows sub-workflow's signature |

The SDK method stays short for code rhythm; the canvas restores semantic clarity. **This is a contract**: any future short method names follow the same pattern.

---

## 6. Out of scope for v0.1

Everything below is deferred. Customer demand from real usage triggers promotion to v0.2.

### Sugar (will land if usage patterns prove demand)

- `step.fetch(url, opts)` — sugar over `step.do` + retry
- `step.map(arr, transform)` — sugar over `step.loop`
- `step.filter(arr, predicate)`
- `step.notify(channel, msg)`
- `step.approve({ approver, question, timeout })`
- `step.queue(queueName, msg)`
- `step.emit(event)`

### Agent-era primitives (the differentiator — see open question 7.1)

- `step.llm({ model, messages, ... })`
- `step.agent({ definition, input, tools?, maxTurns? })`
- `step.tool(name, fn)`
- `step.embed(text)`
- `step.search({ index, query, k })`
- `step.rerank(items, query)`
- `step.ask({ to, question, schema })`

### Advanced control flow

- `step.race(...lanes)`
- `step.any(...lanes)`
- `step.while({ condition, body, maxIterations })`
- `step.try({ try, catch, finally? })`
- `step.timeout({ ms, body })`

### Rejected (do not add — design anti-patterns)

- `step.state.get/set` (mutable scratch breaks deterministic replay)
- `step.memo` (redundant with `step.do`)
- `step.if` / `step.switch` as separate methods (folded into `step.route`)
- `step.fork` (conflated with `step.all`)
- `step.exec(jsString)` (defeats type safety)
- `step.invoke` (naming-clash with `step.wf`)
- `step.cancel` self-cancel (use `step.stop`)

### Parallel namespace (not in this SDK)

`wf.*` is a separate SDK (different import surface) for workflow lifecycle ops from outside a workflow definition: `wf.trigger`, `wf.cancel`, `wf.list`, `wf.deploy`, `wf.events`, `wf.versions`, `wf.sendEvent`. See companion spec.

---

## 7. Open questions (deferred decisions, not blockers)

### 7.1 Agent-era primitives — v0.1 or v0.2?

Strong case for `step.llm`, `step.agent`, `step.tool`, `step.ask` in v0.1 because they're the deck's differentiator. Risk: design isn't fully locked yet. Recommendation: target v0.2 GA but ship as experimental in v0.1 (`@typenode-ai/sdk/experimental` subpath).

### 7.2 `step.do` name

Locked as `step.do` (matches Cloudflare convention; user decision). Alternative `step.task` (matches Zapier/n8n/Step Functions) was considered. Revisit only if first 10 customers ask for it.

### 7.3 Naming for `step.meta` properties

Locked: `runId`, `workflowId`, `workflowVersion`, `tenantId`, `attempt`, `traceId`, `deployedAt`. Add `parentRunId` if/when sub-workflow run tree introspection lands.

### 7.4 ESLint rule package

Ship `eslint-plugin-typenode` in v0.1 with these rules:
- `no-non-deterministic-globals`: bans `Date.now()`, `Math.random()`, `crypto.randomUUID()` inside `step.do(fn)` bodies
- `no-outer-scope-capture-in-expr`: bans variable capture in `step.expr(_, compute)` closures
- `step-do-name-unique`: warns when two `step.do` calls in the same scope share a name

Required for v0.1 GA, not blocking for v0.1 alpha.

### 7.5 Synth-time error messages

When the build function throws (e.g., undeclared dependency in `step.expr`), the error must point to the source line in the customer's `.workflow.ts`. Requires source maps through the synthesize machinery. Track separately.

---

## 8. Frozen acceptance criteria

This memo is locked when:

- [x] All 17 method signatures fit the type machinery in §3 with no ad-hoc extensions
- [x] Every operator has a canvas label specified (§2 + §5)
- [x] Every operator has a compile target named for CF, Vercel, ASL, Inngest (§2)
- [x] All deferred items are explicitly listed (§6) — nothing in v0.1 by default
- [x] Open questions are explicitly named, not silently held

If you find yourself adding a method not on this list while implementing v0.1, **stop and update this memo first.** No silent surface growth.
