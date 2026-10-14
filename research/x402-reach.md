# Can a stock x402 client pay a TOON connector route today?

> Research note for [issue #10](https://github.com/toon-protocol/toon-agent-native/issues/10)
> (map: [#1](https://github.com/toon-protocol/toon-agent-native/issues/1)). It decides nothing.
> Written 2026-10-14. Answered from code, not only from ADR intent. Terms are
> [CONTEXT.md](../CONTEXT.md)'s and TOON's
> [glossary](https://github.com/toon-protocol/toon-meta/blob/main/context/glossary.md)'s.

## The answer

**No.** Over neither HTTP nor MCP, with neither `exact` nor `batch-settlement`. An unmodified
x402 client cannot get a TOON connector to deliver a request to an app. The maintainer has also
**decided it will not be added**: connector issue
[#1442](https://github.com/toon-protocol/connector/issues/1442) was closed `wontfix` on
2026-10-01 under the rule **"a payment is a packet"**.

ADR 0075's line "A stock x402 client can pay any TOON connector" (decision 10, still in the text
at `539dd8f6`) holds only for the **greeting's terms**. A stock client can *parse* the greeting
and build a channel the connector would admit. It cannot *carry* a payment there, and it cannot
*open* the channel the way it expects to. #1442 says so itself and lists 0075 among the records
"still read the old way".

The x402 *channel* and *voucher signature* are compatible. Everything around them is not:

| x402 stock client expects (HTTP transport v2, `batch-settlement` EVM) | TOON connector does (client edge, `539dd8f6`) |
|---|---|
| Calls the **resource's own URL** with its normal method and body | Only `POST /ilp` (plus `GET /ilp`, `/ilp/btp`, `/ilp/probe`, `/ilp/identity`, `/ilp/routes/price`, `/ilp/claim-state`). Routes are keyed by **ILP address prefix**, never by URL path or `Host` |
| Unpaid request gets `402` + `PAYMENT-REQUIRED` | Yes, but only if the body **decodes as a TOON-dialect ILP PREPARE**. Anything else is `400` before the greeting is considered |
| Pays with `PAYMENT-SIGNATURE: base64(PaymentPayload)` (`x402Version`, `accepted`, `payload{type, channelConfig, voucher, deposit?}`) | Reads only `ILP-Payment-Channel-Claim` / `-Wrapped`: a **flat** TOON JSON (`version`, `scheme`, `blockchain`, `messageId`, `timestamp`, `senderId`, `channelId`, `maxClaimableAmount`, `signature`, `channelConfig?`). `PAYMENT-SIGNATURE` appears nowhere in the connector's code |
| First payment is `type: "deposit"`, which the **seller** relays to its facilitator | The connector **never** calls a facilitator (ADR 0076 §1). The payer must post the deposit itself, to `extra.facilitator` (EVM, a TOON-only field) or `sponsorEndpoint` (Solana) |
| Seller returns `PAYMENT-RESPONSE` with `extra.chargedAmount` and `channelState`. The client advances its running total only from that | No `PAYMENT-RESPONSE`. The answer is a binary FULFILL/REJECT. A client that re-signs the same cumulative amount is refused as non-advancing (§1.3 freshness) |
| A failed request leaves the voucher unspent. A corrective `402` resyncs after state loss | A voucher is spent on admission (ADR 0042). Resync is the owner-authenticated `POST /ilp/claim-state` |
| Request and response in plaintext | The payload must be **NIP-59 gift-wrapped** to the connector's key, and the answer is sealed (ADR 0018). `deliver_to_app` always opens a wrap first |
| `exact` scheme | Not offered and not accepted: `accepts[]` holds only `batch-settlement` entries, and any other `scheme` in a claim is malformed |
| MCP: `tools/call` with `_meta["x402/payment"]` | The connector has **no MCP surface at all** (no `mcp` in `crates/`) |

**So what is missing on the client edge is not one header but a second architecture.** It would
need a URL-to-route mapping, an unsealed termination (or sealing on the client's behalf, which
makes the edge a plaintext reader), a deposit relay, a `PAYMENT-RESPONSE` with charged amount, a
corrective-402 resync, and unspent-on-failure semantics. #1442 names this "the second architecture
with its own payment verification" that ADR 0022 warned about, and rejects it.

**For the map:** an **outside agent with a stock x402 client cannot reach a TOON-fronted
Agent-Native app**. An outside payer needs TOON code: `@toon-protocol/client`, the `toon` CLI, or
its own agent node (the direction #1442 points to,
[toon-client#715](https://github.com/toon-protocol/toon-client/issues/715)). The Earning
Direction's payers are therefore limited to agents that already run TOON. That cuts against
"real paid traffic" unless something outside the connector bridges.

## Sources pinned

| Tag | Source | Pin |
|---|---|---|
| **CONN** | [`toon-protocol/connector`](https://github.com/toon-protocol/connector) `main`, fresh clone | `539dd8f6` (2026-10-14). The local checkout was at `48a9db38`, 35 commits behind, and was not used for code claims |
| **X402** | [`x402-foundation/x402`](https://github.com/x402-foundation/x402) `specs/`, `typescript/packages/mechanisms/evm` | `e95b0666` (2026-10-13). `coinbase/x402` now stops at `dd927a26` (2026-04-21), so the foundation repo is canonical |
| **#1442** | [connector#1442](https://github.com/toon-protocol/connector/issues/1442), maintainer decision (closed `wontfix` 2026-10-01; it read `main@8b938f9b` and x402 `@6b6ee91f`) | read 2026-10-14 |

## 1. What the connector accepts (code)

- **Router.** `crates/connector-client-edge/src/lib.rs:415-420` mounts `/ilp` (POST packet, GET
  self-description), `/ilp/btp`, `/ilp/probe`, `/ilp/identity`, `/ilp/routes/price`,
  `/ilp/claim-state`. No other client path exists. Routes are keyed by `prefix`
  (`crates/connector-config/src/route.rs:70`), an ILP address.
- **Decode before greet.** `handle_ilp` (`lib.rs:1187`) runs peer-role dispatch, then `ILP-Peer-Id`
  auth, then `Prepare::decode(&body)` and returns `400` on failure (`lib.rs:1233-1236`). Only after
  that does it check for an absent claim and answer the `402` greeting (`lib.rs:1238-1249`). The
  PREPARE is TOON's dialect, not RFC 0027 bytes: no outer wrapper, a VarUInt amount, and a 19-byte
  `expiresAt` (`docs/protocol/client-edge-spec.md` §1.1, ADR 0063).
- **Claim header.** `CLAIM_HEADER = "ilp-payment-channel-claim"`
  (`crates/connector-btp/src/frame.rs:81`), `CLAIM_WRAPPED_HEADER =
  "ilp-payment-channel-claim-wrapped"` (`lib.rs:132`), read in `extract_and_validate_claim`
  (`lib.rs:982-988`). A grep for `payment-signature`, `x-payment` and `x402/payment` across
  `crates/` and `packages/` finds no reader. The only x402 header the connector *writes* is
  `Payment-Required`.
- **Claim shape.** `crates/connector-domain/src/client_claim.rs`: `EvmVoucher` /
  `SolanaVoucher` with `ClientClaimCommon {message_id, timestamp, sender_id}`. The field *names*
  follow x402's voucher (`channelId`, `maxClaimableAmount`, `signature`, `channelConfig`), but the
  envelope is TOON's own. There is no `x402Version`, `accepted`, `payload.type` or `deposit`.
- **Termination is sealed.** `deliver_to_app` (`crates/connector-runtime/src/connector.rs:3525`)
  calls `open_termination_request(&prepare.data)` (the ADR 0018 wrap) before calling the app, and
  `seal_termination_response` after it.
- **Greeting.** `client-edge-spec.md` §1.4: the body is an x402 v2 `PaymentRequired`, repeated
  base64 in `Payment-Required`. `accepts[]` holds only `batch-settlement` entries. `resource.url`
  is the ILP address (e.g. `g.example.app`). TOON's terms sit in `extensions.toon`, with
  `endpoint: "/ilp"`.

## 2. What a stock x402 client does (spec and SDK)

- **HTTP transport** (`specs/transports-v2/http.md`): `402` + `PAYMENT-REQUIRED`, then a retry of
  the same request with `PAYMENT-SIGNATURE: base64(PaymentPayload)`, answered with
  `PAYMENT-RESPONSE`. "All x402 protocol information is communicated through headers" (l.174).
- **`batch-settlement` EVM** (`specs/schemes/batch-settlement/scheme_batch_settlement_evm.md`):
  - The payload `type` is `deposit`, `voucher` or `refund` (§Client: Payment Construction).
  - "Deposits are sponsored by the facilitator", and the deposit payload goes to the **server**,
    which calls facilitator `/verify` and `/settle` (§Server: Request Processing). The server "must
    not update voucher state until the resource handler has succeeded".
  - The voucher ceiling must equal `chargedCumulativeAmount + amount` exactly, and a mismatch gets
    a corrective 402.
  - The `PAYMENT-RESPONSE` carries `extra.chargedAmount` and `extra.channelState`.
- **SDK** (`typescript/packages/mechanisms/evm/src/batch-settlement/client/channel.ts:50-60`): it
  defaults `payerAuthorizer` to the voucher signer or the payer, which is **nonzero**. So a stock
  client's channel *would* pass TOON's nonzero-`payerAuthorizer` admission rule (spec §1.4), and
  it requires a nonzero `extra.receiverAuthorizer`, which TOON publishes. The core schema needs
  only a non-empty `resource.url` (`typescript/packages/core/src/schemas/index.ts:72`), so an ILP
  address there parses. **The channel and the greeting are compatible. The carrier is not.**
- **MCP transport** (`specs/transports-v2/mcp.md`): a `tools/call` answer with `isError: true` and
  `PaymentRequired` in `structuredContent`, then a retry with `_meta["x402/payment"]`, answered with
  `_meta["x402/payment-response"]`. The connector serves no MCP.
- **`exact`**: one on-chain transfer per call. TOON never offers it.

## 3. The maintainer's decision (#1442)

#1442 asked whether ADR 0022's deferral of "paying over plain HTTP" should end, now that a voucher
does not settle per request. Decision: **no**. "The connector will not accept a voucher on a plain
HTTP request, with or without x402's `PAYMENT-SIGNATURE` header." Its reasons match §1 and §2
above: no route key, no length to price, no expiry, and no unsealed termination on a plain
request. A stock client also expects the four seller behaviours (deposit relay,
`PAYMENT-RESPONSE`, unspent-on-failure, corrective 402).

It considered **x402 headers on the packet path** (`POST /ilp`, sealed body, voucher in
`PAYMENT-SIGNATURE`) and the edge **sealing on a client's behalf**, and took neither. The need was
redirected to "the agent runs its own node", mapped at toon-client#715.

At `539dd8f6`, ADR 0022 (`:104`, "deliberately deferred, not rejected") and ADR 0075 (`:313`,
`:452`) were still unedited, as #1442's "Left undone" says.

## 4. Consequences for this map

- **Earning (connector in front of an Agent-Native Action):** only TOON-equipped agents can pay.
  The large x402 client population (1,738 MCP Registry entries mention x402, per the
  payment-options note) cannot. An Overlap that counts on "outside agents" must either accept a
  TOON-client-only payer base or put a bridge **outside** the connector.
- **TOON vs plain x402 per call** (issue #4's comparison): the "Stock MCP / A2A clients: no" row
  is confirmed from code, and it is now a decided position, not a gap awaiting work.
- **A bridge outside the connector** is not ruled out by #1442, which binds only the connector. It
  would be an x402 seller: a service that relays deposits, verifies `PAYMENT-SIGNATURE`, and pays
  onward from its own TOON node, the way a gateway agent would. It is, in effect, the "gateway
  agent" already in the map's *Not yet specified*, with the same economics question. Nothing like
  it exists in the org (a search for `PAYMENT-SIGNATURE` across toon-protocol finds only research
  notes).

## Not checked

- No live run: no stock x402 client was pointed at a running connector. The conclusion rests on
  code paths, which are unambiguous at the `400`/no-header-reader steps.
- Solana `batch-settlement` client flow in the SDK (the EVM one was read). The connector side's
  carrier gap is chain-independent.
- toon-client#715 / #717 (the agent-node map) beyond #1442's summary of them.
