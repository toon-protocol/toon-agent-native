# Does Agent-Native already have a billing, payment or metering story?

Answers [issue #3](https://github.com/toon-protocol/toon-agent-native/issues/3). Researched 2026-10-14 against
`BuilderIO/agent-native` at commit `3ab5193` (main, 2026-10-14) and the docs at
https://www.agent-native.com/docs/ (the docs site is built from `packages/core/docs/content/*.mdx` in the same repo,
so the file paths below are the source of the pages).

## Answer

**No.** Agent-Native has no way for an app author to charge, or bill per call, the outside callers of an Action. Nothing in
the framework, templates, capability packages, Dispatch, Public Agent Web, or Production Agent Access takes money from
a caller. There is no Stripe Checkout, no paywall, no x402 and no HTTP 402 *issued* to callers. The money-shaped code
that does exist points the other way: it tracks and caps **what the app spends on LLM inference**, and Builder.io
sells that inference as **Builder.io credits**. Rate limits are small, hard-coded guards against abuse, not quotas per
caller. So there is nothing for TOON to compete with on the Earning side. TOON would fill an empty slot rather than
replace something. The nearest seams to plug into are the existing bearer-token caller identities (org service tokens,
A2A JWT) and the `publicAgent` opt-in on Actions.

Licence: **MIT** (Copyright (c) Builder.io). Capability packages and extensions are distributed as **npm packages**
(`@agent-native/*`) installed with the `agent-native package` CLI, plus a shadcn-style `registry.json`. Extensions are
sandboxed iframe mini-apps stored inside an app and shared within an org. There is no marketplace and nothing is paid.

## Findings

### 1. No caller-side charging anywhere

- Searching the whole repo for `x402` finds nothing. Every `402` in `packages/core/src` is the framework *receiving* a 402 from
  Builder.io's hosted gateway when the app's own credits run out (`agent/engine/builder-engine.ts:634`,
  `transcription/builder-transcription.ts:105`, `server/builder-design-systems.ts:924`, `server/transcribe-voice.ts:386`).
  None of them is a 402 sent to a caller.
- Stripe shows up only as **something the agent can use**, not as billing for the app: a CLI adapter example
  (`docs/content/cli-adapters.mdx`), an MCP integration listed under "Payments & billing" (`integrations.mdx:125,346`),
  a `provider stripe` blueprint (`blueprint-installer.mdx:20`), and the Analytics template's read-only revenue routes
  (`templates/analytics/server/routes/api/stripe/{billing,subscriptions,refunds,payment-status,billing-by-product}.get.ts`),
  which chart the user's own Stripe data.
- The docs mention "charge a card" only as an example of a consequential action that should set `needsApproval: true`
  (`actions-access-control.mdx:20`, `embedding-sdk.mdx:652`, `durable-resume.mdx:103`).
- `README.md`, `PRODUCT.md` and `AGENTS.md` contain no billing, pricing or monetisation plans.
- A search of upstream issues for billing, stripe, payment, monetize or x402 turned up nothing relevant (two unrelated hits).

### 2. Metering exists, but only for LLM spend the app pays for

- **Usage store** `packages/core/src/usage/store.ts` records token usage and estimated cost for each model call. It defines
  `UsageBillingMode` with units `usd | builder-credits | mixed`, and the Builder credit rate:
  `BUILDER_AGENT_CREDIT_MARGIN_MULTIPLIER = 1.25` and `BUILDER_AGENT_CREDITS_PER_USD = 20`. The Observability, Toolkit
  Observability and Dispatch "Admin → Metrics" pages show this as "spend" per app, model, user and org
  (`observability.mdx:29`, `toolkit-observability.mdx`, `template-dispatch-features.mdx:308`,
  `packages/dispatch/src/server/lib/usage-metrics-store.ts`).
- **Builder.io credits.** By default, Builder.io's hosted gateway supplies the LLM, transcription and design-system calls,
  and it bills the *app owner* in credits. The free tier includes some credits (`template-chat-developers.mdx:50`,
  `template-clips-features.mdx:191,203`). When credits run out the framework emails the owner a `core.builder-credit-limit` notice
  (`usage/builder-credit-notice.ts`, `deployment.mdx:118`). The upgrade link goes to
  `https://builder.io/account/subscription` (`shared/builder-link-tracking.ts:43`). This is Builder.io's own business
  model: it sells inference to app authors. It is not something those authors can resell to callers.
- **Spend budgets.** `packages/core/src/integrations/usage-budget-store.ts` holds daily or monthly budgets in
  `currency_micros` for an org, a user or an integration scope. The flow is reserve, then settle or release (`reserveIntegrationUsageBudget`,
  `settleIntegrationUsageBudget`, `releaseIntegrationUsageBudget`). Messaging (Slack and similar) uses these budgets for
  each org, requester and channel: "reserved before a run and settled from actual usage" (`messaging.mdx:199-203`). This is
  the closest thing to a payment primitive in the codebase, and it is a **spend cap** on the app's own LLM cost, not a charge
  to the caller.

### 3. Rate limits are abuse guards, not per-caller quotas

These are all small in-memory or SQL counters on specific endpoints: `/generate-title` allows 10 calls per 60 s per user
(`server/agent-chat-plugin.ts:6469`), the Builder waitlist allows 5 per 60 s (`server/core-routes-plugin.ts:1324`), MCP
connect returns 429 (`mcp/connect-route.ts:1630`), and messaging allows 20 inbound messages per hour per sender
(`messaging.mdx:376`, `messaging-internals.mdx:209`). There is no configurable quota per token or per caller on Action calls.

### 4. The surfaces outside agents call, and how callers are identified

- **Production Agent Access** (`actions-agent-tools.mdx`) controls which framework tools and code execution the deployed
  agent has. It does not meter or charge anything.
- **HTTP API** (`http-api.mdx`): every `defineAction` is a `POST /_agent-native/actions/<id>` endpoint that takes
  `Authorization: Bearer`. Tokens are either personal or **org service tokens**
  (`svc-<name>@service.<orgId>`, 1–365 day TTL, can be suspended or revoked as service principals). Callers are identified, but
  not billed.
- **A2A** (`a2a-protocol.mdx`): JSON-RPC at `/_agent-native/a2a`, with a JWT bearer signed with `A2A_SECRET` (or a legacy
  `apiKeyEnv`). Hosted runtimes refuse to run without the secret. The agent card advertises only the `jwtBearer` / `apiKey` security
  schemes and has no payment field.
- **MCP** (`mcp-protocol.mdx`): OAuth discovery through `WWW-Authenticate`, or static `ACCESS_TOKENS`.
- **Public Agent Web** (`agent-web-surfaces.mdx`): robots.txt, llms.txt, Markdown mirrors and JSON-LD. Setting
  `publicMcp: true` (off by default) exposes Actions that opt in with `publicAgent: { expose, readOnly,
  requiresAuth, isConsequential }` (`packages/core/src/action.ts:250`). The code allows only Actions that are
  **unauthenticated, read-only and not consequential** (`server/action-routes.ts:525`
  `isPublicWebMcpAction`, `server/agent-chat/action-filters-a2a.ts:66` `filterPublicAgentActions`). So the one
  surface open to anonymous agents is free by construction, and nothing on it can be paid for.
- **Dispatch** (`dispatch.mdx`, `template-dispatch-*.mdx`) is the workspace control plane: it holds shared secrets
  (an example given is "Stripe key"), approvals, an MCP gateway, and usage metrics. It does not bill callers.

### 5. Licence and distribution

- Root `README.md` ends with "## License — MIT". There is no root `LICENSE` file, but `packages/vscode-extension/LICENSE.md` is the
  MIT text, "Copyright (c) Builder.io". Every `packages/*/package.json` says `"license": "MIT"`. The root
  `package.json` (private workspace) says `ISC`. npm shows `@agent-native/core@0.208.0` and
  `@agent-native/scheduling@0.2.27` as MIT.
- **Capability packages** ("Toolkit capability modules", `toolkit-capability-packages.mdx`): separate npm packages
  (`@agent-native/scheduling`, `@agent-native/creative-context`, `@agent-native/pinpoint`), each with a static
  `agent-native.package.json` manifest. The `agent-native package inspect | add | eject` commands are dry-run unless given `--apply`
  (`package-lifecycle.mdx`). They are all first-party, and there is no third-party catalogue or payment.
- **Templates** ("Cloneable SaaS", `cloneable-saas.mdx`): fork a complete app and deploy it as your own product. If the
  author wants to charge, they wire that up themselves. The framework gives no help.
- **Extensions** (`extensions.mdx`): sandboxed iframe mini-apps inside a template, private by default and shareable
  org-wide or per user. There is no marketplace.
- **Registry**: root `registry.json` uses the shadcn registry schema to ship conventions and skills files.

## What this means for the map

- **Earning**: no incumbent. A connector in front of the HTTP/A2A/MCP endpoints would be the *first* way to charge
  outside agents per Action. The Public Agent Web path cannot carry paid calls without changes, because public
  Actions must be unauthenticated, read-only and not consequential, and nothing in the code calls out to a payment check.
  Wrapping it from outside (a reverse proxy) needs no upstream change.
- **Paying**: Agent-Native's own payment flow goes to Builder.io (credits for LLM inference). Paying TOON apps would sit
  next to it, not replace it.
- x402: Agent-Native does not support it, so "TOON packets vs plain x402" is a choice about something not yet built, not a
  fight with an existing integration.

## Sources

- Repo: https://github.com/BuilderIO/agent-native (commit `3ab5193312fd8519675bf19846dec5c298258619`)
- Docs: https://www.agent-native.com/docs/ (pages cited by slug, e.g. `/docs/agent-web-surfaces/`, `/docs/http-api/`,
  `/docs/a2a-protocol/`, `/docs/messaging/`, `/docs/toolkit-capability-packages/`, `/docs/package-lifecycle/`,
  `/docs/extensions/`, `/docs/actions-agent-tools/`)
- npm: `npm view @agent-native/core license` and `npm view @agent-native/scheduling license`
