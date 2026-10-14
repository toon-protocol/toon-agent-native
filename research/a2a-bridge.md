# A2A and Dispatch vs. TOON's NIP-90 job market

Research for [#6](https://github.com/toon-protocol/toon-agent-native/issues/6) (map: [#1](https://github.com/toon-protocol/toon-agent-native/issues/1)). Direction: **Reaching**. Checked 2026-10-14 against Agent-Native `main` at `3ab5193` and the A2A spec on `a2aproject/A2A@main`.

## Answer

**No bridge exists on either side today, and Agent-Native has no place a payment could go.** Its A2A stack has no payment concept, no A2A extension support, and authenticates callers only by bearer JWT or a static API key. So an Agent-Native agent cannot pay a TOON job, and cannot even be asked to (a payment-required reply), without upstream changes.

You can still build a bridge with **no upstream changes**, but only as an **adapter agent that owns a TOON wallet**. It sits on one side of the line, and the other side never learns TOON exists:

- **A2A → NIP-90 ("gateway agent")**: an external A2A peer that an Agent-Native workspace adds with a bearer token. It turns an A2A message into a `kind:5097` brief and pays the increments from **its own** agent node. It returns the decrypted artifact as an A2A artifact. TOON gets real paid traffic, but the gateway operator fronts the money and must bill the Agent-Native user somewhere outside TOON (the bearer token is the account). This is really **Paying** with a reseller in the middle.
- **NIP-90 → A2A ("provider wrapper")**: a TOON factory provider that does the work by calling an Agent-Native app's A2A endpoint, using a bearer token the app's operator gave it. Outside agents pay through the job market. This is really **Earning** (the app author earns), with the job market as the storefront.

Neither one gives TOON reach into A2A as a network. Dispatch only discovers **workspace siblings**. External peers are wired by hand, one bearer token at a time. Public Agent Web does not advertise A2A services. So "Reaching" turns into one of the other two Directions plus an adapter, and it should rank below them.

## Agent-Native side

### A2A as implemented

- JSON-RPC 2.0 at `POST /_agent-native/a2a`, with `/a2a` as a fallback for external peers. The agent card is public at `/.well-known/agent-card.json`. Methods: `message/send` (optionally `async: true` + poll), `message/stream`, `tasks/get`, `tasks/cancel`. Source: [A2A Protocol docs](https://www.agent-native.com/docs/a2a-protocol/).
- Task states: `submitted → working → completed | failed | canceled | input-required`. Tasks persist in the `a2a_tasks` SQL table (same page).
- **Auth is identity, not payment.** In production, hosted runtimes refuse unauthenticated calls with a 503. Callers present a JWT signed with `A2A_SECRET` (or an org-scoped secret), or a legacy static key named by `apiKeyEnv`. Auth runs at the HTTP boundary, before the agent loop. A static key "does not establish Agent-Native caller identity" (same page; `packages/core/src/a2a/auth-policy.ts`, `server.ts`).
- **No A2A extensions support.** `AgentCapabilities` in `packages/core/src/a2a/types.ts` has only `streaming`, `pushNotifications`, `stateTransitionHistory`, `connect` and `extendedAgentCard`. There is no `extensions` array, and nothing in `packages/core/src/a2a/` reads or sends the `A2A-Extensions` header. Outbound message metadata goes through `sanitizeA2ACorrelationMetadata` (`correlation.ts`), which keeps only correlation fields (`callerApp`, `parentRunId`, `delegationDepth`, …), so the client would never send payment metadata.
- The client treats `input-required` as **terminal** in `sendAndWait` (`client.ts` ~L789). It also maps a peer's `auth-required` to `input-required` (`client.ts` ~L1698). Default poll timeout is 5 min (`client.ts` L734/767/797). Messaging-inbound continuations are capped at 20 min of remote work and 30 attempts (`integrations/a2a-continuation-processor.ts` L78; docs "Continuations").
- A grep of the whole repo for `x402`, `ap2` or `paymentRequired` finds nothing in core. The only hits are the word "payments" as a triage category in `templates/factory`.

### Dispatch

- Dispatch is the workspace control plane: one inbox for Slack, email, Telegram and WhatsApp, a vault, cross-app delegation, an MCP gateway and resources. "Dispatch auto-discovers the other apps in your workspace as A2A peers", using the workspace registry and auto-signed per-app credentials. Source: [Dispatch docs](https://www.agent-native.com/docs/dispatch/); `packages/core/src/server/agent-discovery.ts`.
- External A2A "still uses the bearer-token model (`apiKeyEnv` + `A2AClient(url, apiKey)`). Workspace A2A is layered on top" ([A2A docs, Workspace A2A](https://www.agent-native.com/docs/a2a-protocol/)). Dispatch has no market, no RFQ and no price. It routes by intent to a known sibling.

### Public Agent Web and External Agents

- Public Agent Web means crawlable public pages: robots, sitemap, `llms.txt`, Markdown twins, JSON-LD and a `publicAgentCard` flag. It also offers an opt-in `publicMcp` for actions marked `publicAgent.expose`. "A public page is not a public tool." Source: [Public Agent Web](https://www.agent-native.com/docs/agent-web-surfaces/). It says nothing about pricing or paid discovery.
- External Agents covers MCP connectors (Claude, ChatGPT, Codex, Cursor) reaching the app via OAuth. A2A is mentioned only as the app-to-app path. Source: [External Agents](https://www.agent-native.com/docs/external-agents/).

### The A2A spec itself

- A2A has a general **extension** mechanism. Agents declare extension URIs in `AgentCard.capabilities.extensions`. Clients opt in with the `A2A-Extensions` HTTP header and put extension data under message `metadata` (spec §4.6, §14.2.2, [specification.md](https://github.com/a2aproject/A2A/blob/main/docs/specification.md)).
- The **a2a-x402 extension** ([google-agentic-commerce/a2a-x402 spec v0.1](https://github.com/google-agentic-commerce/a2a-x402/blob/main/spec/v0.1/spec.md)) is the closest thing to a paid A2A task:
  - The merchant replies `input-required` with `x402.payment.status: payment-required` and payment requirements in message metadata.
  - The client sends back `payment-submitted` with a signed payload.
  - The merchant settles on chain and completes the task with a receipt.
- **Agent-Native implements none of this** (see above). In TOON vocabulary, x402 would only carry a Funding Authorization. The unpaid answer is a connector **greeting**, not a 402 (toon-meta `context/glossary.md`; connector `CONTEXT.md` "Greeting").

## TOON side

- The factory market is NIP-90 on a relay ([factory-job-protocol.md](https://github.com/toon-protocol/toon-meta/blob/main/docs/factory-job-protocol.md)). It uses three kinds:
  - `kind:5097`: the brief, with `bid` as a ceiling in micro-USDC and `param repo/target/constraints`.
  - `kind:7000`: feedback, one of `quote`, `partial` (the increment offer) or `processing` (narration).
  - `kind:6097`: the result, with `outcome` ∈ {completed, abandoned-provider, abandoned-buyer}.
- **Discovery is pull, on purpose.** Providers subscribe to `kind:5097`. There is **no `kind:31990`** advertisement for factory jobs (§1.3). A2A is the opposite model: you call a known URL whose card you already fetched.
- **There is no accept message. Paying increment 1 is the acceptance** (§3.3).
- **Delivery is by hashlock.** The `condition` tag equals the ILP PREPARE `executionCondition`, and the FULFILL's `fulfillment` is the artifact's decryption key. The PREPARE `data` field carries the `kind:7000` event id (§4.2). The buyer therefore needs a connector and a funded channel, which means an agent node.
- Increments are milestones, not tokens (§4.3). A job can run for hours across plan → implement×N → review. That is far longer than Agent-Native's 5-min default poll and 20-min continuation cap.
- Private briefs (NIP-59 gift wrap) work only for **targeted** briefs (§2.2). A2A is always targeted, so an A2A-originated job maps naturally to a targeted brief, or to an open RFQ that the gateway runs itself.
- No TOON doc mentions A2A (`grep` across `toon-meta/context`, `toon-meta/docs`, `toon_cli/nips`).

## Concept mapping

| A2A (Agent-Native) | NIP-90 factory (TOON) | Gap |
|---|---|---|
| Agent card at a known URL, fetched before calling | Pull: post `kind:5097`, anyone may quote; no `kind:31990` | A2A needs a target; the market needs no target. A gateway hides the RFQ behind one card. |
| `message/send` | `kind:5097` brief (`i` = text, `p` = targeted provider) | Brief needs `bid` (ceiling) — A2A has no price field. |
| `working` + status messages | `kind:7000 status:processing` (narration) | Clean map. |
| `input-required` | `kind:7000 status:quote` / `partial` (pay to continue) | Agent-Native treats `input-required` as terminal and surfaces only text, so there is no payment handshake. |
| Artifact (inline text/data/file) | Encrypted Arweave blob + key released in ILP FULFILL | The gateway must pay, fetch and decrypt, then re-emit the artifact inline. |
| `completed` / `failed` / `canceled` | `kind:6097 outcome` completed / abandoned-* | Clean-ish map. Reputation comes only from 6097. |
| Bearer JWT / static key (identity) | Payment claim in ILP packet (who paid, how much) | Different trust bases. The bridge must turn one into the other, off-protocol. |
| Dispatch: route by intent to a workspace sibling | Open market of unknown providers | Dispatch never leaves the workspace without a hand-wired peer. |

## What each side would have to understand

- **For an Agent-Native agent to buy on the job market directly**, it would need: a TOON agent node and wallet; the ability to post a priced brief and read quotes; a way to pay increments with a matching `executionCondition`; and the patience to fetch and decrypt Arweave artifacts across a long-running job. None of this is in Agent-Native. A TOON skill or action inside one *app* could do it (app code, not upstream). But then the app is a TOON client (the **Paying** Direction), and A2A plays no part.
- **For A2A itself to carry the payment**, Agent-Native would need A2A extensions support (card `capabilities.extensions`, the `A2A-Extensions` header, metadata passthrough) plus a client that acts on a payment `input-required`. That is upstream work, so at best a **conditional go**.
- **For a TOON provider to use Agent-Native as its worker**, it needs only an A2A endpoint URL and a bearer token from the app's operator. All the TOON machinery (quote, encrypt, Arweave, hashlock, 6097) stays in the provider. This works today with no upstream changes.

## Ranking input for the Fit Verdict

1. **Provider wrapper (NIP-90 → A2A)**: no upstream changes, and real paid traffic comes from existing job-market buyers. Credit it under **Earning**, not Reaching.
2. **Gateway agent (A2A → NIP-90)**: no upstream changes on Agent-Native. Paid traffic reaches TOON providers, but it is funded by the gateway operator, who must charge Agent-Native users outside TOON. Long jobs clash with A2A timeouts. Weak.
3. **Native paid A2A** (a2a-x402-style or a TOON greeting extension): needs upstream A2A extension support, so a conditional go at best.

Beats plain x402 per call? For A2A specifically, neither TOON nor x402 has a slot in Agent-Native today, so the comparison is moot until upstream adds extensions. Inside the factory market, TOON's hashlock (payment and key handoff in one packet, milestone increments) does something an a2a-x402 per-task payment does not: the a2a-x402 merchant settles, then delivers, which is trust-the-merchant.

## Side note: a name collision

Agent-Native ships a **"Factory"** template (`templates/factory`: "a workspace of named factories: Inbox, automations, and a reviewable map"). It is unrelated to TOON's factory job market. Name it carefully in CONTEXT.md and in the verdict.
