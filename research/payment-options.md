# What payment options already exist for A2A and MCP calls?

> Research note for [issue #4](https://github.com/toon-protocol/toon-agent-native/issues/4)
> (map: [#1](https://github.com/toon-protocol/toon-agent-native/issues/1)). It decides nothing.
> Written 2026-10-14. Primary sources only, each pinned below. Terms are
> [CONTEXT.md](../CONTEXT.md)'s and TOON's
> [glossary](https://github.com/toon-protocol/toon-meta/blob/main/context/glossary.md)'s. In TOON
> vocabulary x402 is only the wire standard a Funding Authorization rides. Where this note says
> "x402" alone, it means the outside standard as its own spec defines it.

## The answer

**One family dominates, and Agent-Native supports none of it.**

- **x402 is the base layer under almost everything.** It has normative transport bindings for
  **MCP** and **A2A** in its own repo, and four schemes: `exact` and `upto`, which settle on chain
  per call, plus `batch-settlement` (prepaid channel, vouchers per call) and `auth-capture`.
  Repo active daily, about 6.7k stars, under the x402 Foundation.
- **The A2A x402 extension** (`google-agentic-commerce/a2a-x402`) is the A2A-side spec of the same
  flow, at v0.2. Its only release is v0.1.0 (2025-09), and the Python library is `0.1.0`.
  **Early.**
- **AP2 is not a per-call payment rail.** It is a mandate (authorization and dispute evidence)
  layer for shopping-agent checkouts, at v0.2 (2026-04). It uses x402 as one "form of payment"
  through a2a-x402's embedded flow. It has nothing to say about paying per tool call.
- **MCP itself defines no payment.** SEP-2007 closed dormant on 2026-06-24, and the follow-up
  proposal (#3393) closed `NOT_PLANNED` on 2026-09-28. Payment on MCP lives outside the spec: x402's
  MCP transport (`_meta["x402/payment"]`), Cloudflare's `paidTool`, and **MPP** (Stripe and Tempo,
  IETF-track). MPP's MCP transport (`-32042`) and its **payment proxy for an existing service** are
  the closest thing to TOON's pitch on the market.
- **Agent-Native: no support for any of them.** Its MCP and A2A endpoints authenticate with
  OAuth, a JWT or a bearer token. Its generated agent card has no `extensions` field, so it cannot
  declare a2a-x402 without an upstream change. In its code, "402" only means the LLM gateway's
  credit limit.

**Against a TOON packet, plain x402 per call comes in two forms, and TOON beats only one of them
outright.** TOON's channels **are** x402 `batch-settlement` channels (connector ADR 0075). So against
x402 `batch-settlement` sold directly, the per-call economics are **the same**: one deposit, a
signed voucher per call, no gas per call, settlement off the request path. TOON's remaining edge is
**multi-hop**. One channel to one connector reaches any app behind any peering. A plain x402 channel
is payer-to-seller only. Against x402 `exact` (the default in every MCP/A2A example), TOON wins
clearly on per-call cost and latency. TOON's cost is that no stock MCP or A2A host can speak its
client edge today.

## Sources pinned

| Tag | Source | Pin |
|---|---|---|
| **X402** | [`x402-foundation/x402`](https://github.com/x402-foundation/x402) `specs/` | `e95b0666` (2026-10-13) |
| **A2AX** | [`google-agentic-commerce/a2a-x402`](https://github.com/google-agentic-commerce/a2a-x402) | `125db552` (2026-05-24) |
| **AP2** | [`google-agentic-commerce/AP2`](https://github.com/google-agentic-commerce/AP2) `docs/` | `e1ea56db` (2026-04-29) |
| **AN** | [`BuilderIO/agent-native`](https://github.com/BuilderIO/agent-native), docs source `packages/core/docs/content/` (served at <https://www.agent-native.com/docs/>) | `3ab51933` (2026-10-14) |
| **MCP** | [`modelcontextprotocol/modelcontextprotocol`](https://github.com/modelcontextprotocol/modelcontextprotocol) PR/issue tracker; latest spec release `2026-07-28` | read 2026-10-14 |
| **MPP** | <https://mpp.dev> (`/protocol/transports/mcp`, `/intents/session`, `/guides/proxy-existing-service`, `/governance`) | read 2026-10-14 |
| **CF** | [Cloudflare "Charge for MCP tools"](https://developers.cloudflare.com/agents/tools/payments/x402/charge-for-mcp-tools/) | last updated 2026-06-03 |
| **ADR** | [`toon-protocol/connector`](https://github.com/toon-protocol/connector) `docs/adr/` | `48a9db38` |
| **CRES** | connector `docs/research/mcp-servers-for-toon.md` (prior TOON research, 2026-10-01) | `48a9db38` |
| **META** | [`toon-protocol/toon-meta`](https://github.com/toon-protocol/toon-meta) `context/`, `docs/payment-proxy.md` | `b3603954` |

Repo activity was read from the GitHub API on 2026-10-14.

---

## 1. x402: the base layer

**What it is.** HTTP 402 plus a `PaymentRequired` object (`accepts[]`: scheme, network, amount,
asset, `payTo`) and a signed `PaymentPayload` sent back on retry. A **facilitator** verifies and
settles on the seller's behalf (X402 `specs/x402-specification-v2.md`). The default flow is
`authorization`: verify, then the resource runs, then settle, then respond. `upfront` settles
before the resource runs (§6.1, lines 287-299).

**Schemes** (X402 `specs/schemes/`):

| Scheme | Per call | Notes |
|---|---|---|
| `exact` | one on-chain transfer per call (EIP-3009 on EVM), via the facilitator | 17 network bindings (EVM, SVM, Aptos, Stellar, TON, Lightning, …) |
| `upto` | one on-chain transfer per call, variable amount up to a signed max | for metered calls such as LLM tokens; single use per authorization |
| `batch-settlement` | **no on-chain transfer per call**: a signed commitment is stored and redeemed later | *capital-backed* (EVM/SVM payment channel, `scheme_batch_settlement_evm.md`) or *credit-backed* (Cloudflare `cloudflare:402`, an RFC 9421-signed identity billed off chain) |
| `auth-capture` | escrowed authorization then capture | EVM |

`batch-settlement` on EVM (`scheme_batch_settlement_evm.md` §Summary, §Channel Lifecycle): a
deposit creates a unidirectional channel, with the deposit sponsored by the facilitator so it is
gasless for the client. The client signs a **cumulative voucher** per request. The server checks it
with a signature check and claims many channels in one transaction later. The channel is keyed by
`ChannelConfig {payer, receiver, …}`, so it is **one payer to one receiver**.

**MCP binding** (X402 `specs/transports-v2/mcp.md`). An unpaid `tools/call` returns a tool result
with `isError: true` and `PaymentRequired` in `structuredContent`. The client retries the same
`tools/call` with `_meta["x402/payment"]`. The server verifies, runs the tool, settles, and returns
`_meta["x402/payment-response"]` holding the transaction hash. It is used in the wild: CRES counted
1,738 MCP Registry entries mentioning x402 and saw one live server answer exactly this shape (CRES
§(d)).

**A2A binding** (X402 `specs/transports-v2/a2a.md`). The task goes to `input-required` with
`x402.payment.status: "payment-required"`. The client sends a new message on the same `taskId` with
`x402.payment.payload`. The status then moves through `payment-submitted`, `payment-verified` and
`payment-completed`, and the receipts arrive in `x402.payment.receipts`. The agent must declare the
extension in its AgentCard, and the client activates it with the `X-A2A-Extensions` header.

**Maturity: high for `exact` over HTTP, moderate for MCP/A2A.** The MCP and A2A bindings are short
specs, and every example in them uses `scheme: "exact"`. The repo has about 6.7k stars and was pushed
2026-10-14. The x402 Foundation is under the Linux Foundation (META `docs/payment-proxy.md`).

## 2. The A2A x402 extension

A2AX `spec/v0.2/spec.md` has the extension URI `…/a2a-x402/blob/main/spec/v0.2`. The x402 repo's own
A2A binding still names the v0.1 URI, `https://github.com/google-a2a/a2a-x402/v0.1`, so the two
disagree on which URI to declare. The extension has two flows (§4):

- **Standalone**: x402 objects ride in task and message metadata, the same as X402's A2A binding.
- **Embedded**: the x402 objects nest inside a higher protocol's objects (an AP2 `CartMandate` /
  `PaymentMandate`), "x402 acting as a Form of Payment".

Settlement is on chain per payment ("Supports payments using the x402 protocol for on-chain
settlement", §3). There is no session or channel mode in the spec. It also lists experimental
partner schemes (`schemes/`: Lightning, Spark, UMA).

**Maturity: early.** It has one release, v0.1.0 (2025-09-16). The library `x402-a2a` is `0.1.0`
(Coinbase and Google authors, `python/x402_a2a/pyproject.toml`), Python only. The repo has 564
stars and was last pushed 2026-08-04.

## 3. AP2: mandates, not metering

AP2 `docs/ap2/specification.md` (v0.2) defines roles (Shopping Agent, Credential Provider,
Merchant, Merchant Payment Processor, Trusted Surface), a **Checkout Mandate** and a **Payment
Mandate** as signed evidence of *what* was bought and *that* payment was authorized, and two modes:
human-present, and human-not-present under open mandates with constraints. "AP2 operates as a
security feature within a Commerce Protocol"; catalog and checkout APIs are out of scope.
Agent-to-agent delegation is "outside the scope of the current specification". AP2 `docs/faq.md`
points to a2a-x402 for stablecoin payment.

**Relevance to per-call payment: none directly.** AP2 adds a signed mandate around a checkout. It
does not move value and does not price a tool call. **Maturity:** v0.2.0 (2026-04-28), about 3.2k
stars, samples on ADK/Gemini, SDK in progress (FAQ).

## 4. MCP: no payment in the spec

- **SEP-2007, "Payment Support for MCP Servers"** (draft, sponsor LucaButBoring): capability
  declaration, pricing in `tools/list`, error `-32402`, x402 and fiat. Closed 2026-06-24 without
  merging. Duplicates #2008 and #2009 also closed.
- **#3229** "Native Token Metering, Session Budgets, and Payment Tracking (x402)" was closed
  2026-08-23. **#3393** "paid MCP tools: a standard client-side payment flow (x402)?" was closed
  `NOT_PLANNED` on 2026-09-28.
- No open SEP about payment was found by search on 2026-10-14. Adjacent open work: SEP-2752 (HTTP
  Message Signing for client auth) and SEP-2643 (Structured Authorization Denials).

**Payment on MCP is done outside the spec, in three ways:**

1. **x402's MCP transport**, described above.
2. **Cloudflare Agents SDK `paidTool`** (CF): wrap `McpServer` with `withX402`, set a USD price per
   tool, on `base` / `base-sepolia` through `x402.org/facilitator`. That is x402 `exact`.
3. **MPP, the Machine Payments Protocol** (MPP `/governance`): co-authored by Tempo and Stripe. Its
   core is "the Payment HTTP Authentication Scheme", submitted to the IETF. On MCP, an unpaid call
   gets JSON-RPC error `-32042` with `challenges`, the credential rides in
   `_meta["org.paymentauth/credential"]`, and the receipt in `_meta["org.paymentauth/receipt"]`
   (MPP `/protocol/transports/mcp`). It has a **`session` intent**, "reusable payment state" over
   payment channels (Tempo, Solana, Lightning, Stellar) that is "best … across many MCP tool calls"
   and for sub-cent pricing (MPP `/intents/session`). It also has a **proxy that gates an existing
   API without changing it**, injecting the upstream credential (MPP
   `/guides/proxy-existing-service`), which is TOON's "payment as a reverse proxy" with a different
   wire. Its docs list x402 compatibility middleware. Stripe documents a 0.01 USDC minimum for
   stablecoin machine payments (CRES caveats).

## 5. Agent-Native: what it supports today

- **MCP server**: every app auto-mounts `POST /mcp` (Streamable HTTP). Auth is standard remote-MCP
  OAuth 2.1, a connect-minted JWT, or static `ACCESS_TOKEN(S)`. An unauthenticated request gets `401`
  plus `WWW-Authenticate` (AN `mcp-protocol.mdx` §Authentication).
- **A2A server**: `POST /_agent-native/a2a`, card at `/.well-known/agent-card.json`, JWT bearer
  signed with `A2A_SECRET` or a legacy API key. A hosted runtime with no secret **fails closed** (AN
  `a2a-protocol.mdx`).
- **No payment protocol.** The docs and code contain no x402, AP2 or MPP. The only `402` handling
  maps an LLM gateway's "credits limit" to a stop error (AN
  `packages/core/src/client/error-format.spec.ts:86`,
  `packages/core/src/agent/engine/builder-engine.spec.ts:897`). Stripe appears only as a data or
  MCP integration.
- **The A2A card cannot declare an extension.** `generateAgentCard` emits `protocolVersion: "0.3"`,
  `capabilities {streaming, pushNotifications, stateTransitionHistory, connect}`, `skills` and
  security schemes, and no `extensions` (AN `packages/core/src/a2a/agent-card.ts`). Declaring
  a2a-x402 therefore needs an upstream change or an override of the card route.
- **An escape hatch exists:** Nitro middleware in `server/middleware/` runs on every request,
  "HTTP · MCP · A2A" (AN `server-middleware.mdx`). An author could add x402 or MPP gating there by
  hand. Nothing is shipped for it.

**So an Agent-Native author who wants per-call revenue today would reach for**: Cloudflare-style
x402 `exact` on the MCP endpoint through custom middleware, or an MPP proxy in front of it. Neither
is built in.

---

## 6. Comparison per call

The TOON packet as the connector defines it today (ADR 0042, 0061, 0065, 0074, 0075, 0076; META
`context.md`, `architecture.md`): a PREPARE carries its own **covering claim**, which is now always an
x402 `batch-settlement` **voucher** on an x402 channel (ADR 0075: "every channel is an x402 channel").
The channel is opened once through **Onboarding**: a **Funding Authorization** relayed by the
facilitator the operator names, with the operator paying its gas (ADR 0076). An unpaid request to a
priced route gets a **greeting** with the terms, and `GET /ilp` returns the self-description. Each
hop is a peering that mints its own covering claim for the forwarded value and takes a flat
per-packet **fee** (ADR 0061). Settlement is an operator write, never on the packet path.

| | x402 `exact` (MCP/A2A default) | x402 `batch-settlement` (EVM/SVM, direct) | a2a-x402 | MPP `session` | **TOON packet + covering claim** |
|---|---|---|---|---|---|
| **Round trips, first call** | 2 (call → `PaymentRequired`, retry with payload); A2A adds the task/message hops | 2 + deposit (via facilitator) | 2 task/message exchanges | 2 + channel open | 1 greeting or self-description read + Onboarding once per counterparty and chain, then 1 |
| **Round trips, steady state** | 2 per call unless the client caches requirements; the seller also waits on facilitator verify + settle | 1 (voucher with the request) | 2 | 1 (signed update) | **1** (PREPARE → FULFILL/REJECT) |
| **Per-call on-chain cost** | one transfer per call (gas paid by the facilitator; CDP bills past a free tier, ADR 0076) | **none**: batched claim + settle | one transfer per call | none (method-defined) | **none**: vouchers landed by an operator write |
| **Per-call fee** | facilitator terms | none beyond the seller's price | facilitator terms | method and processor terms (Stripe min. 0.01 USDC) | route price + flat **fee per peering crossed** |
| **Prepaid vs pay-as-you-go** | pay-as-you-go, no prepayment | prepaid escrow, metered by voucher | pay-as-you-go | prepaid session | **prepaid channel**, metered by voucher |
| **Multi-hop** | no: `payTo` is the seller | **no**: a channel is payer-to-receiver | no | not in the protocol (MPP lists "Relays" as an advanced feature; not read) | **yes**: one channel to one connector reaches any app behind any chain of peerings; each hop is paid on arrival by its own covering claim (ADR 0042), and the fulfilment is an end-to-end delivery receipt, not a hashlock (ADR 0069) |
| **Stock MCP / A2A clients** | yes (x402 libs, Cloudflare SDK) | yes (x402 libs) | Python lib | yes (`mppx`) | **no**: needs `@toon-protocol/client` or the `toon` CLI; no crate reads `PAYMENT-SIGNATURE` or MCP `_meta` (CRES caveats) |
| **App stays payment-oblivious** | no, unless wrapped (middleware) | no, unless wrapped | no | yes, with the MPP proxy | **yes** (the connector is the proxy) |

**What the table says, plainly:**

1. **TOON clearly beats x402 `exact`** per call: one round trip against two, and no chain write
   against one per call. `exact` is what every MCP and A2A example, Cloudflare's `paidTool` and
   a2a-x402 use.
2. **TOON does not beat x402 `batch-settlement` on per-call economics.** It uses the same contract,
   the same deposit and the same voucher. A seller who runs x402 `batch-settlement` directly gets the
   same one-round-trip, zero-gas call. TOON's edge over it is **multi-hop reach**, plus the reverse
   proxy that keeps the app payment-oblivious, which MPP's proxy also offers.
3. **TOON's cost is client reach.** A stock MCP host with x402 or MPP support can pay a Cloudflare
   or MPP-gated tool today. It cannot pay a TOON route without a TOON client. That cuts against the
   Earning Direction, whose payers are outside agents.

## 7. Facts the later tickets will run into

- **An Agent-Native endpoint is not keyless.** `/mcp` and `/_agent-native/a2a` require a bearer,
  JWT or OAuth token in production, and a TOON route cannot add a header to what it delivers (CRES
  "Buckets", CONFIRMED-CODE). A connector in front of an Agent-Native app therefore needs a small
  app between them that holds the token, or an author-set static `ACCESS_TOKEN` that the *payer*
  would have to send, which defeats permissionless payment (ADR 0052). This bears on the Earning
  ranking.
- **The two A2A x402 URIs disagree** (v0.1 `google-a2a/...` in X402 against v0.2
  `google-agentic-commerce/...` in A2AX). A client has to accept both.
- **ADR 0075's "a stock x402 client can pay any TOON connector"** holds for the greeting's terms,
  not for the carrier (CRES caveats; flagged there, unresolved).
- **META `docs/payment-proxy.md` is partly stale.** It still calls the unpaid answer a `402` with a
  `toon-channel` entry. ADR 0075 deleted `toon-channel`, and the glossary retires "402" in favour of
  "greeting".
- **META `context.md` is stale on atomicity.** It still says "multi-hop atomicity is packet-level
  condition/fulfilment". ADR 0069 removed the execution condition from the wire. Under ADR 0042 a
  hop is paid on arrival, and only the sender checks the fulfilment.

## Not checked

- MPP's "Relays" feature (it might give multi-hop) and its exact session wire per method.
- Live behaviour of any paid x402 or MPP MCP endpoint. Nothing was paid.
- AP2's v0.2 x402 sample (`a2a-x402/python/examples/ap2-demo`), which was not run.
