# Does MPP give multi-hop payment?

> Research note for [issue #11](https://github.com/toon-protocol/toon-agent-native/issues/11)
> (map: [#1](https://github.com/toon-protocol/toon-agent-native/issues/1)). It decides nothing.
> Written 2026-10-14. Primary sources only, each pinned below. Terms are
> [CONTEXT.md](../CONTEXT.md)'s and TOON's
> [glossary](https://github.com/toon-protocol/toon-meta/blob/main/context/glossary.md)'s. Follows
> [research/payment-options.md](https://github.com/toon-protocol/toon-agent-native/blob/research/payment-options/research/payment-options.md)
> (issue #4), which left "Relays" unread.

## The answer

**No. MPP has no multi-hop payment at the protocol layer.** Every MPP payment is between one
client and one server (the challenge's `realm` and `recipient`). Nothing in the spec or the SDK
forwards a payment across a chain of independent operators who each keep a fee.

- **"Relays" are not hops.** A relay is a back-office service *behind* the seller. It validates a
  Credential and broadcasts the settlement transaction for the seller's own payment. It never
  receives or passes on value, and the client never sees it. The spec does not define relays at
  all; the docs say so in as many words (MPP `/advanced/relays`, `/blog/relays`).
- **The payment proxy is one hop.** `mppx/proxy` gates an upstream API and injects a static
  upstream API key (bearer, header, or `rewriteRequest`). It has no notion of paying the upstream.
  It is TOON's "payment as a reverse proxy" for a single edge (MPP `/sdk/typescript/proxy`).
- **`session` is a two-party channel.** Tempo, EVM, Solana and Stellar sessions are
  *unidirectional* channels from a `payer` to a `payee`, and the payee must equal the challenge's
  `recipient` (`draft-tempo-session-00`, `draft-evm-session-00`). A channel cannot be routed through.
- **Two things look like multi-hop and are not.** *Split payments* send one `charge` to up to 10
  recipients in one atomic on-chain transaction. That is fan-out from the payer, not a path, and it
  exists for `charge` only (MPP `/guides/split-payments`). *Lightning methods* ride a network
  that is multi-hop underneath (HTLC routing, onion routing), but that is the rail's routing
  between the payer's and the server's Lightning nodes. MPP sees one payer and one payee
  (MPP `/payment-methods/lightning/*`, `draft-lightning-charge-00`, `draft-lightning-session-00`).

**So TOON's multi-hop edge survives against MPP**, as it survived against x402
`batch-settlement`. A TOON payer opens one channel to one connector and reaches any app behind any
chain of peerings, and each peering keeps a flat fee (connector ADR 0042, 0058, 0061). MPP gives a
payer no way to reach a seller it has no channel or payment method with, except by paying that
seller directly.

**One caveat, which narrows the edge.** Two MPP proxies *can* be chained by hand: the custom
`fetch` option of `Proxy.create` could be an MPP-paying client, so proxy A pays proxy B out of its
own wallet. That is two unrelated payments with A carrying the risk, no fee convention, no
discovery of the path's cost, and no end-to-end delivery receipt. It is a reseller, not a route.
(Inference from the documented `fetch` option; not built or tested.)

## Sources pinned

| Tag | Source | Pin |
|---|---|---|
| **MPPD** | <https://mpp.dev> docs, Markdown form of each page: `/advanced/relays`, `/blog/relays`, `/intents/session`, `/sdk/typescript/proxy`, `/guides/proxy-existing-service`, `/guides/split-payments`, `/payment-methods/tempo/session`, `/payment-methods/lightning/{charge,session}`, `/mpp-vs-x402`, `/overview`, `/llms.txt` | read 2026-10-14 |
| **SPEC** | [`tempoxyz/mpp-specs`](https://github.com/tempoxyz/mpp-specs) `specs/` (the IETF-track drafts rendered at <https://paymentauth.org>) | `50309c8` (2026-10-05) |
| **ADR** | [`toon-protocol/connector`](https://github.com/toon-protocol/connector) `docs/adr/` 0042, 0058, 0061, 0069 (and 0010, 0011, 0028 they amend) | `48a9db38` (2026-09-30) |
| **META** | [`toon-protocol/toon-meta`](https://github.com/toon-protocol/toon-meta) `context/` | `b360395` (2026-10-01) |

---

## 1. What the spec says about anything between payer and payee

The core draft is `draft-httpauth-payment-01`, "The 'Payment' HTTP Authentication Scheme",
category `std` (SPEC `specs/core/`).

- **A challenge is bound to one server.** "Servers MUST bind the challenge `id` to `realm`,
  `method`, `intent`, and …" and must verify that binding on the credential (§Challenge Binding,
  lines 359-373). A credential is good only at the realm that issued it, so it cannot be handed
  on to a next hop.
- **The only intermediaries it mentions are HTTP ones.** §"Intermediary Handling of 402"
  (lines 1053-1063) is about proxies, caches and CDNs that may not treat 402 like 401. "Servers
  SHOULD NOT rely on intermediary-specific handling." Nothing about payment intermediaries.
- **A search of the whole drafts tree** for relay, intermediary, hop, proxy, onward and downstream
  (core, intents, extensions, Tempo, EVM, Lightning) finds only: the above; caching
  intermediaries in the subscription drafts; "intermediary operations such as token swaps" inside
  one atomic Tempo `charge` transaction (`draft-tempo-charge-00` line 323); and the EVM session's
  **"Relayed / Gasless Operations profile"** (`draft-evm-session-00` §relayed-profile), where a
  "relayer" only submits a channel open, settle or close transaction and pays its gas. That is
  ADR 0076's "the operator pays the facilitator's gas", not routing.
- **There is no session intent draft.** `specs/intents/` holds only `charge` and `subscription`;
  `session` is defined per method (Tempo, EVM, Solana, Hedera, Lightning). The docs agree:
  "Method-specific session implementations define the exact `request` and `payload` fields"
  (MPPD `/intents/session`).

## 2. Relays

MPPD `/advanced/relays`:

> Relays sit between your application and the payment-method settlement layer, giving your
> application a consistent payment-method-agnostic interface.

The sequence diagram is `Client → API → (402) → Client → API + Credential → Relay: validate →
Relay: finalize → Relay → settlement layer: broadcast → Receipt → API → Client`. Two hooks:
`validate` (no state change, no funds reserved) and `broadcast` (submit to the rail). "MPP does not
define a relay API shape" and "The MPP spec does not formally define relays." The default relay is
Tempo's own API (`https://api.tempo.xyz`), keyed by the seller's `TEMPO_API_KEY`.

The blog post announcing them (MPPD `/blog/relays`) frames the purpose as "abstracts payment-method
settlement away from your application." Its closest TOON analogue is the **facilitator**, not a
peering.

## 3. The payment proxy

MPPD `/sdk/typescript/proxy` and `/guides/proxy-existing-service`: `Proxy.create({ services })`
mounts each upstream at `/{serviceId}/`, prices routes with `mppx.charge(...)` or
`mppx.tempo.session(...)`, marks free routes `true`, and authenticates upstream with an injected
credential (`Authorization: Bearer`, `x-api-key`, Basic, a `headers` map, or `rewriteRequest`).
Presets exist for OpenAI, Anthropic and Stripe. Nothing in it splits the payment with the upstream
or pays the upstream through MPP. The `fetch` option ("Custom `fetch` implementation") is the only
hook that could, as the caveat above says.

## 4. Sessions are two-party

- **Tempo** (`draft-tempo-session-00`, MPPD `/payment-methods/tempo/session`): "a unidirectional
  streaming payment channel" on the TIP-1034 precompile. The channel descriptor has `payer`,
  `payee`, an optional payee-side `operator`, and an `authorizedSigner`. The server checks
  "`descriptor.payee` matches the challenge `recipient`". Cumulative vouchers, batched settlement,
  server-owned `settlementSchedule`.
- **EVM** (`draft-evm-session-00`): the same shape, "unidirectional streaming payment channels for
  incremental, voucher-based" payment, `payee` = "Server authorized to withdraw".
- **Lightning** (`draft-lightning-session-00`, MPPD `/payment-methods/lightning/session`): not a
  channel at all at the MPP layer. The client pays a deposit invoice, and the preimage becomes a
  bearer token against a server-side balance, refunded to a return invoice on close.

This is the same economics as an x402 `batch-settlement` channel and as a TOON channel: one
deposit, a voucher per call, no gas per call. Like x402's, it ends at the payee.

## 5. Split payments

MPPD `/guides/split-payments`: "Split a single charge across multiple recipients in one atomic
transaction." 1 to 10 splits, each `> 0`, sum strictly less than `amount`; the primary recipient
gets the remainder. The client can pin `expectedRecipients`. The examples are a platform fee and a
seller, referrals, revenue shares. Only `charge` is shown; nothing extends splits to `session`. It
pays several parties for one purchase. It does not let a payer reach a seller through someone else.

## 6. The spec tracker

A search of `tempoxyz/mpp-specs` issues and PRs (2026-10-14) for multi-hop, hop, routing,
intermediary and relay finds **no multi-hop proposal**, open or closed. Two items are relevant:

- **PR #192 "Add facilitator extension for payment authentication"**, closed unmerged
  2026-03-22: a `POST /verify` + `POST /settle` intermediary for the seller. That is what shipped,
  informally, as relays (section 2), and it confirms relays are a facilitator, not a hop.
- **Issue #311** (open, 2026-08-02): "Lightning forwarding nodes can reuse deposit preimages to
  access paid sessions." In `draft-lightning-session-00` the deposit preimage is the session's
  bearer token, and every Lightning forwarding node on the route learns it during HTLC settlement.
  So the one place MPP touches real multi-hop routing, the hops are an attack surface, not paid
  participants.

## 7. TOON's multi-hop, for comparison

From the connector ADRs (protocol law where so scoped):

- **Every packet carries its own covering claim**, at every hop, "whether that connector will
  forward it or terminate it; and a connector covers every PREPARE it sends" (ADR 0042, protocol
  law). A hop is paid on arrival.
- **A fulfilment is "a delivery receipt, not a payment trigger"** (ADR 0042). ADR 0069 (protocol
  law) removed the execution condition from `Prepare` entirely. No hop verifies anything; the
  sender alone compares the returned fulfilment with `derive_fulfillment` of its own sealed secret.
  The risk a hop takes the claim and drops the packet is bounded by the per-peer cap and by the
  sender's packet sizing, not by a hashlock (ADR 0042, "The trade this makes").
- **A peering is one operator write from a URL**: `POST /peers { id, url, fee, max_packet_amount }`
  reads the peer's self-description, derives the channel and opens it (ADR 0058, amended by 0075:
  each side funds its own outbound x402 channel).
- **Fees are flat, per packet, per peering**; a route's `price` buys the whole path and each hop
  keeps `fee`, forwarding `price − fee` (ADR 0061, amending 0010 and 0028). Rejects accumulate
  fees, so a probe discovers a path's cost (ADR 0011).

MPP has no counterpart to any of these four: no forwarded value, no per-hop fee, no path-cost
discovery, no end-to-end delivery receipt across operators.

## Facts the later tickets will run into

- **The edge is real but narrow.** Multi-hop matters only if a payer wants to reach sellers it has
  no channel with. Against MPP, as against x402, a seller that runs MPP directly gives a stock
  client the same one-round-trip, zero-gas call per voucher. TOON's case for the leading Overlap
  therefore has to rest on *reach through peerings* (or the reverse-proxy shape, which MPP's proxy
  matches), and has to say who the intermediate peers would be.
- **MPP's proxy plus a TOON connector is a possible bridge.** Both are reverse proxies that keep the
  app payment-oblivious. An MPP-paying client could enter at a TOON edge that accepts an MPP
  credential and covers the onward packets itself. Nothing like this exists; it is the same open
  question as ADR 0075's "a stock x402 client can pay any TOON connector".
- **META `context/context.md` line 49 and `context/decisions.md` line 40 are stale.** Both still
  say "Multi-hop atomicity is packet-level condition/fulfilment". Under ADR 0042 and 0069 there is
  no multi-hop atomicity: hops are paid on arrival and only the sender checks the fulfilment. Any
  verdict text that claims TOON multi-hop is *atomic* would be wrong. (Also flagged in the issue #4
  note.)

## Not checked

- Live behaviour of any MPP relay, proxy or session. Nothing was paid.
- Whether `Proxy.create`'s `fetch` accepts an `mppx` client fetch in practice (the chained-proxy
  caveat). Not built.
- Hedera, Solana, Stellar and XRPL session drafts in detail. They were skimmed only for the same
  terms; nothing suggested they differ from Tempo and EVM on the payer/payee shape.
- Issues and PRs in MPP repos other than `tempoxyz/mpp-specs` (for example the `mppx` SDK).
