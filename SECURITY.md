# Security Policy

## Reporting a Vulnerability

If you discover a security vulnerability in Typenode SDK, please report it privately.

**Do not open a public GitHub issue.**

Email: **security@typenode.ai**

Include:
- A description of the vulnerability
- Steps to reproduce (if applicable)
- The affected package and version
- Your assessment of severity

We will acknowledge receipt within 2 business days and provide a more detailed response within 7 days indicating next steps. We aim to keep you informed of progress toward a fix and full announcement.

## Scope

This policy covers:
- `@typenode-ai/sdk`, `@typenode-ai/core`, `@typenode-ai/cli`, `@typenode-ai/eslint-plugin`
- `@typenode-ai/provider-*` packages
- Code in this repository

Out of scope:
- Vulnerabilities in vendor backends (Cloudflare, Vercel, AWS, Inngest) — report to those vendors
- Vulnerabilities in customer code that uses the SDK — that's their concern

## Supported Versions

Until v1.0.0, only the latest published version receives security updates. Pin to specific versions if you need stability guarantees.

## Disclosure

After a fix is released, we'll publish a security advisory through GitHub Security Advisories and credit the reporter (unless you prefer to remain anonymous).
