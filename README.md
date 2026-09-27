# Typenode SDK

Public, Apache-2.0 SDK design repository for portable TypeScript workflows. **This is a scaffold,
not an implemented SDK or published deployment CLI.** The package directories contain manifests
and design notes; they currently have no `src/` implementations. Scripts in those manifests are
placeholders.

## Current contents

- [`docs/architecture.md`](docs/architecture.md) describes the planned TypeScript source, typed
  intermediate representation, and provider compiler model.
- [`docs/methods-v0.1.md`](docs/methods-v0.1.md) records the proposed v0.1 method surface.
- [`packages/`](packages/) reserves package identities for the SDK, core, CLI, lint rules, and
  Cloudflare, Vercel, and AWS Step Functions providers.

## Planned developer flow

The intended design is to author a `.workflow.ts` file, synthesize it into a portable IR, then
compile and deploy it to a chosen provider. That `workflow`/`step` API, the `typenode` CLI, and
provider targets are **planned interfaces**. They are not installed, built, or deployable from
this repository today. The Cloudflare provider is the first intended implementation target;
Vercel and AWS follow as portability checks.

Read the [architecture](docs/architecture.md) and [methods](docs/methods-v0.1.md) for design
feedback. Typenode's private coding workspace includes this repository as the future SDK home.
See [CONTRIBUTING.md](CONTRIBUTING.md) and [LICENSE](LICENSE) for contribution and license terms.
