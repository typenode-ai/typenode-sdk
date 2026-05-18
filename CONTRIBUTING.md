# Contributing to Typenode SDK

Thank you for your interest in Typenode SDK. We're in early design — feedback on the architecture and method surface is the highest-leverage contribution right now.

## Where we are

The SDK is **pre-release**. The architecture and v0.1 method surface are locked, but no production code has shipped yet.

- Architecture: [`docs/architecture.md`](./docs/architecture.md)
- Method surface (v0.1): [`docs/methods-v0.1.md`](./docs/methods-v0.1.md)

## What's most useful right now

**Feedback on the design memos.** Open a [GitHub Discussion](https://github.com/typenode-ai/typenode-sdk/discussions) or issue if you have:

- Concerns about the synthesize-time-vs-runtime model
- Use cases that the locked operator surface doesn't cover
- Vendor-portability questions (does the IR survive the target you care about?)
- Pushback on naming, ergonomics, or any specific operator
- Real-world workflows you'd want to express — share them as code samples and we'll see if they fit

## What's NOT useful yet

- Pull requests adding code — the packages are scaffolds with no implementation. Wait for the first real release.
- Proposals to add operators beyond the v0.1 surface — see §6 of `docs/methods-v0.1.md` for the deferred list. We'll revisit after v0.1 ships.

## When code does land

Once we're past v0.1 alpha, contributions will follow a standard flow:

1. Open an issue describing the change before writing code
2. Fork, branch, change, PR
3. CI runs typecheck + lint + tests
4. We review with the design memos as the contract

## License

By contributing, you agree your contributions are licensed under [Apache-2.0](./LICENSE).

## Code of Conduct

Be kind, be specific, assume good faith. We'll publish a formal Code of Conduct alongside the first real release.

## Questions?

- GitHub Discussions: https://github.com/typenode-ai/typenode-sdk/discussions
- Email: hello@typenode.ai
