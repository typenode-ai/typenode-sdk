# @typenode-ai/cli

Command-line tool for Typenode workflows. Pulumi-style lifecycle commands.

**Status:** Scaffold — no implementation yet.

## Will provide

```bash
typenode init                          # Scaffold a new workflow project
typenode synth <file>.workflow.ts      # TS → IR
typenode diff <id>                     # Diff committed IR vs current synthesis
typenode validate <file>               # Type-check + structural validation
typenode dev                           # Local dev: synth + run + canvas viewer
typenode deploy --target=cloudflare    # Compile + deploy
typenode deploy --target=vercel
typenode deploy --target=aws-sfn
typenode runs list <id>                # List recent runs
typenode runs inspect <run-id>         # Show run timeline + step IO
```

See [docs/architecture.md](../../docs/architecture.md) for the full CLI design.
