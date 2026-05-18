# @typenode-ai/provider-aws-sfn

Compiles Typenode IR to AWS Step Functions ASL JSON + per-leaf Lambda function artifacts.

**Status:** Scaffold — third compiler target (portability proof). No implementation yet.

## Will provide

- `compile(ir, opts) → AwsSfnDeployment` — emits ASL JSON + per-leaf Lambda packages
- Maps each Typenode IR op to its ASL state:
  - `do` → `Task` state (Lambda invoke)
  - `route` → `Choice` state (JSONata conditions)
  - `all` → `Parallel` state
  - `loop` → `Map` state
  - `pause` → `Wait` state (duration)
  - `listen` → `Wait` state + task token callback
  - `stop` → `Fail` state
  - `output` → `Pass` state + `End: true`
  - `wf` → `Task` with `StartExecution.sync2`

The cleanest IR portability test — ASL has no JS escape hatch, so expressions must compile to JSONata or fall back to an evaluator Lambda. If this target compiles cleanly without ad-hoc IR extensions, the architecture is validated.
