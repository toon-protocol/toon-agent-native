# Can each paying agent get its own identity in an Agent-Native app?

Research for [#9](https://github.com/toon-protocol/toon-agent-native/issues/9) (map: [#1](https://github.com/toon-protocol/toon-agent-native/issues/1)). Builds on [#2](https://github.com/toon-protocol/toon-agent-native/issues/2) (`research/payment-entry.md` on branch `research/payment-entry`). Terms as in [CONTEXT.md](../CONTEXT.md).

Sources are pinned:
- Agent-Native: [`BuilderIO/agent-native@3ab5193`](https://github.com/BuilderIO/agent-native/tree/3ab5193312fd8519675bf19846dec5c298258619) (2026-10-14). Current `main` (`70edcec`, same day) differs only in three slides-template files, so every file cited here is identical on `main`. Docs are the MDX sources behind agent-native.com/docs (`packages/core/docs/content/`).
- TOON connector: [`toon-protocol/connector@539dd8f`](https://github.com/toon-protocol/connector/tree/539dd8f6fcf095cae2e61ba17232a97be9642686) (2026-10-14) and its ADRs.
- TOON payment proxy RFC: `toon-meta/docs/payment-proxy.md` (last changed `a60249d`, 2026-08-28).

`AN:` means `https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/`, and `CN:` means `https://github.com/toon-protocol/connector/blob/539dd8f6fcf095cae2e61ba17232a97be9642686/`.

Everything below comes from reading the code and docs. None of it was run against a live app; the prototype ticket should check the chosen path end to end.

## Answer

**Yes, with no upstream change, on the HTTP action route too, but only if the shim holds a credential that can mint any identity.** There are four ways, and they trade isolation against how much power the shim gets:

1. **The shim signs a per-payer A2A JWT with the app's `A2A_SECRET`** (HTTP action route and `/mcp`). This gives the best isolation: each payer becomes its own user with no org, so it sees only its own rows. The cost is the largest. `A2A_SECRET` is deployment-wide signing material: whoever holds it can act as any user, including the author, and can also sign MCP OAuth access tokens.
2. **The shim sends a static `ACCESS_TOKEN` plus `X-Agent-Native-Owner-Email: <payer>`** (`/mcp` only). Each payer again gets its own user with no org. The token lets the holder act as any user on the full MCP surface, which is narrower than (1). But it only works on `/mcp`, so it inherits #2's flat price per POST. It also needs `AGENT_NATIVE_OWNER_EMAIL` to be unset, and Agent-Native's own comment says multi-tenant deployments must not use it.
3. **The shim mints one org service token per payer** (HTTP action route). It uses an admin token for a dedicated "payers" org. Each payer gets its own `svc-*` identity, so private rows stay separate. But every payer is an implicit `member` of that org and can read its org-visible rows. The cost is admin power over that one org, plus a token to mint and store for each new payer.
4. **The app author adds an `actionRouteAuth.resolveCaller` adapter** that trusts the shim (HTTP action route). This is the author's own app code, not an upstream change. The shim then needs only a narrow shared secret, and the author decides the mapping. The cost is that the author writes code, so it is not zero-touch.

**Whatever the option, the identity is a channel, not a wallet.** `X-TOON-Payer` is the client channel key the claim was admitted under (`evm:0x<64 hex>`, `solana:<base58>`). It is present only when this connector verified the claim itself and the route has a non-zero price. A payer that opens a new channel becomes a new user and loses access to its old rows, unless the shim resolves the channel to its counterparty address.

**If none of these is acceptable, and the shim gets only one shared service token, then the Earning Overlaps that survive are Actions that keep no per-caller state.** That means read-only Actions, stateless computations, and writes whose result every payer may see. Reading other payers' writes is a real cost for anything that stores a payer's data, such as Clips recordings, forms or documents.

## The identity paths in Agent-Native

### How a request gets its identity

Both surfaces end in `verifyAuth(authHeader, ownerEmailHeader, options)` ([AN:packages/core/src/mcp/build-server.ts](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/src/mcp/build-server.ts) line ~4462). It checks, in order:
1. a credential this app issued (MCP OAuth access token or connect token)
2. a static `ACCESS_TOKEN`
3. dev-open
4. an A2A JWT
5. a static token again

- **`/mcp`** passes the `x-agent-native-owner-email` request header as `ownerEmailHeader` ([AN:packages/core/src/mcp/server.ts](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/src/mcp/server.ts) lines 331-371).
- **The HTTP action route** (`/_agent-native/actions/*`) calls `verifyAuth(authHeader, undefined, { allowDevOpen: false, … })` and refuses any result with no `userEmail` ([AN:packages/core/src/server/auth.ts](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/src/server/auth.ts) `getMcpOAuthBearerSession`, lines 1196-1259). Its own comment says it honors "EXACTLY the tokens the MCP endpoint honors". `getBearerSession` reaches it only for framework action routes (lines 1270-1275).

The resulting `{ userEmail, orgId }` becomes the request context, and data scoping keys off it. Rows carry `owner_email`, `org_id` and `visibility` from `ownableColumns()`. Every new resource is **private to its creator by default** and can be widened to `org` or `public` ([AN:packages/core/docs/content/sharing.mdx](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/docs/content/sharing.mdx) "The three visibility levels"). Agent SQL and the access guards filter on those columns ([AN:packages/core/docs/content/security.mdx](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/docs/content/security.mdx) "Data Scoping"; [multi-tenancy.mdx](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/docs/content/multi-tenancy.mdx) "How isolation works"). The docs warn that Postgres RLS is not yet enforced and that these layers "do not replace application authorization checks".

When a token carries no org, the org is looked up from `org_members` by email ([AN:packages/core/src/org/context.ts](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/src/org/context.ts) `resolveOrgIdForEmail`, line 593). A synthetic payer email is in no org, so it runs in **personal scope**: it sees its own rows and public rows, nothing else.

### Option 1: an A2A JWT signed with `A2A_SECRET`

- **Mechanism.** In `verifyAuth`, a bearer that isn't an issued credential is tried as an A2A JWT (`verifyA2AJwtForMcp`). If it carries no `aud`, it is verified with no audience against the global `A2A_SECRET`. If it carries no `org_domain`/`org_id`, no organization check runs. It is admitted with `userEmail: payload.sub` and `identityAssurance: "user"`. The only further gate is `admitServicePrincipal`, which matters only for service-shaped emails (build-server.ts lines ~3988-4078 and ~4500-4530; `parseMcpOAuthOrgIdClaim` in [AN:packages/core/src/mcp/oauth-token.ts](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/src/mcp/oauth-token.ts) line 94 accepts a missing `org_id`). The function's own doc states: "A deployment-secret A2A JWT may supply its asserted user (`sub`)". The static-token comment says "multi-tenant deployments must use A2A JWT (verified `sub`), not a static token, for per-user scope".
- **Docs.** [security.mdx](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/docs/content/security.mdx) "A2A Identity Verification": "App A signs a JWT containing `sub: \"steve@example.com\"` … App B reads the verified `sub` claim into request context … App B only shows Steve's data."
- **Surfaces.** The HTTP action route (per Action, so priced per Action as in #2) and `/mcp`.
- **What the shim does.** It overwrites `Authorization` with `Bearer <HS256 JWT {sub: "<payer-derived>@<reserved domain>", exp: short}>`, signed with `A2A_SECRET`.
- **Access granted to the shim.** All of it, in effect:
  - **Any identity.** `A2A_SECRET` lets the holder assert any `sub`, including the author's and any human user's.
  - **OAuth tokens.** MCP OAuth access tokens are "signed with `A2A_SECRET` when set" ([authentication.mdx](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/docs/content/authentication.mdx) line 372), so the holder can sign those too.
  - **A2A and derived keys.** The holder can call the app's A2A endpoint. In hosted workspace deploys the secret also feeds `BETTER_AUTH_SECRET`, OAuth-state keys and the workspace vault key ([security.mdx](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/docs/content/security.mdx) lines 325, 367, 409).
  - **Sibling apps.** In a multi-app workspace the same secret is mounted into every app ([a2a-protocol.mdx](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/docs/content/a2a-protocol.mdx) line 693).
  - **Docs on it.** [organizations-teams-permissions.mdx](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/docs/content/organizations-teams-permissions.mdx) line 529 calls it "deployment-wide signing material".
- **Isolation.** Best: each payer is in personal scope, as long as the synthetic email isn't in `org_members` and isn't service-shaped (`svc-*@service.<orgId>`).
- **Caveat.** No test in the repo exercises an A2A JWT on the HTTP action route; the path is established by reading `verifyAuth` and `getMcpOAuthBearerSession`.

### Option 2: a static `ACCESS_TOKEN` plus `X-Agent-Native-Owner-Email` (`/mcp` only)

- **Mechanism.** `matchStaticAccessToken` → `deriveStaticTokenIdentity(ownerEmailHeader)` → `{ userEmail: owner, orgDomain: undefined }` with `fullSurface: true` (build-server.ts lines ~3925-3962 and ~4568-4590). The configured `AGENT_NATIVE_OWNER_EMAIL` ([AN:packages/core/src/app-config/auth.ts](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/src/app-config/auth.ts) line 8) always wins over the header. So per-payer identity needs it unset.
- **On the HTTP action route this collapses to one identity.** There `ownerEmailHeader` is `undefined`, so a static token yields either the configured owner or no `userEmail` at all, and no `userEmail` means refused.
- **Upstream's stated intent.** "The header path remains for the single-tenant local-dev install flow … multi-tenant deployments must use A2A JWT … not a static token, for per-user scope." The code doesn't prevent a multi-tenant use, but the authors don't intend it.
- **Access granted to the shim.** It can act as any user on the full MCP surface (every tool, `ask-agent`). It does not get signing keys, browser auth (static tokens "are not browser auth", [authentication.mdx](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/docs/content/authentication.mdx) line 289) or A2A. That is narrower than option 1, but it still covers every user's data reachable through tools.
- **Pricing.** It inherits #2's limit: `/mcp` is one flat price per POST, because the tool name sits in the body the connector never reads.

### Option 3: one org service token per payer, in a dedicated org

- **Mechanism.** An org service token is a connect-minted credential for a synthetic `svc-*@service.<orgId>` identity. It acts as an implicit org `member` and is never inserted into `org_members`. Admins mint it with `create-org-service-token`, which can carry an `allowedActions` grant ([AN:packages/core/docs/content/http-api.mdx](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/docs/content/http-api.mdx) "Lifetime, rotation, and revocation" and "Service principals").
- **What the shim does.** It mints one per new `X-TOON-Payer` the first time it sees it, stores the mapping, and overwrites `Authorization` with that payer's token.
- **Isolation.** Partial:
  - Private rows are separate, because each `svc-*` identity owns its own.
  - Every payer is a `member` of the same org, so `visibility: org` rows are shared across payers and with any humans in that org. Service-token writes are described as "org-scoped: every teammate … sees it" (http-api.mdx, Clips example).
  - A dedicated org with no humans contains the spillover to other payers only.
- **Access granted to the shim.** An admin credential for that org, because minting and revoking are admin-gated. The shim does not get any app-wide secret, and it cannot reach other orgs' data.
- **Overheads.**
  - **A token per payer.** Each one is a 1-3650-day JWT plus a governance record.
  - **A mint before the first call.** Minting costs an extra call on a payer's first request.
  - **Revocation.** Revoking one payer is a single `jti` revocation, which is a real advantage.
- **Not checked.** Whether `create-org-service-token` can be called by a non-interactive admin token over the HTTP action route.

### Option 4: the author's own `actionRouteAuth` adapter

- **Mechanism.** The agent-chat plugin takes `actionRouteAuth?: ActionRouteAuthAdapter`. Its `resolveCaller(event)` runs before the cookie/bearer chain on `/_agent-native/actions/*` and returns `{ owner, orgId, … }`. The org comes only from the adapter's result, never from an ambient cookie ([AN:packages/core/src/server/agent-chat/plugin-options.ts](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/src/server/agent-chat/plugin-options.ts) lines 115-130; [AN:packages/core/src/server/action-routes.ts](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/src/server/action-routes.ts) lines 342-363, 866-880, 1100-1115). The framework already uses it itself: [AN:packages/core/src/feature-flags/a2a-action-route.ts](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/src/feature-flags/a2a-action-route.ts).
- **What the shim does.** It adds a narrow shared secret (an HMAC over the request, say) and passes `X-TOON-Payer` through. The adapter checks the secret and returns `owner = <payer-derived email>`, `orgId = null`.
- **Access granted to the shim.** Only what the author's adapter accepts, which could be "act as a `payer:` identity, nothing else".
- **Cost.** This is app code the author writes and maintains. It's not an upstream change, but it's not "connector in front, no code" either. In a ranking it sits between "no change" and "upstream change".

### Comparison

| Option | Surface | Isolation between payers | Shim can… | Author effort |
|---|---|---|---|---|
| Shared org service token (#2's baseline) | HTTP action route | **none**: one identity | act as one org member | mint one token |
| 1. A2A JWT with `A2A_SECRET` | HTTP action route, `/mcp` | **full** (personal scope) | act as **any user**, sign OAuth tokens, call A2A | hand over the deployment secret |
| 2. `ACCESS_TOKEN` + owner-email header | `/mcp` only (flat price) | **full** (personal scope) | act as **any user** on the full MCP surface | set `ACCESS_TOKEN`, leave `AGENT_NATIVE_OWNER_EMAIL` unset; against upstream's stated intent |
| 3. One service token per payer | HTTP action route | private rows only; org rows shared | administer one dedicated org | create a payers org, hand over an admin token |
| 4. `actionRouteAuth` adapter | HTTP action route | **full**, as the author defines it | only what the adapter accepts | write and maintain an adapter |

## The TOON side: what the shim has to map from

- **What the header names.** A terminating connector states `X-TOON-Payer`, `X-TOON-Amount` and `X-TOON-Chain` "only when it verified that payment itself". `X-TOON-Payer` is "the admitted client channel key — `evm:0x<64 lower-case hex>`, `solana:<base58>`". It "names a **channel**, not a wallet" ([CN:docs/adr/0040-a-verified-payment-is-stated-to-the-app.md](https://github.com/toon-protocol/connector/blob/539dd8f6fcf095cae2e61ba17232a97be9642686/docs/adr/0040-a-verified-payment-is-stated-to-the-app.md); implemented in [CN:crates/connector-runtime/src/attribution.rs](https://github.com/toon-protocol/connector/blob/539dd8f6fcf095cae2e61ba17232a97be9642686/crates/connector-runtime/src/attribution.rs)).
- **When it's absent.** Peer-wire arrivals, forwarded packets, unclaimed requests and zero-price routes carry no payer headers. Those are protocol law.
- **A spoofed header can't get through.** The connector strips a caller's own spelling of those three names on every delivery.
- **The RFC is out of date here.** `payment-proxy.md` still calls the header "`X-TOON-Payer` pubkey" (line ~133). ADR 0040 is the law and says channel key, so the RFC is the stale one.

Consequences for the shim:

- **The identity is per channel.** One agent with two channels, or one that reopens a channel, becomes two Agent-Native users. Collapsing them to one wallet means resolving the channel's counterparty from the connector's channel records. ADR 0040 leaves that to the reader, off the hot path.
- **The key has to become an email.** For example `<hex>@payer.<reserved-domain>`, with `solana:` keys hex-encoded or hashed. A 64-hex local part fits the 64-octet limit. Agent-Native doesn't validate the string on these paths, but a reserved domain keeps it out of `org_members` and away from the `svc-*@service.*` shape.
- **The shim must overwrite, not add.** The connector forwards every other header the payer sent ([CN:crates/connector-runtime/src/app_client.rs](https://github.com/toon-protocol/connector/blob/539dd8f6fcf095cae2e61ba17232a97be9642686/crates/connector-runtime/src/app_client.rs)). A payer could otherwise send its own `X-Agent-Native-Owner-Email` (option 2) or `Authorization` and pick any identity. The shim must replace both, and drop the request when `X-TOON-Payer` is absent.
- **The hop must be private.** As `payment-proxy.md` says, "the backend trusts injected headers … over a private hop". With options 1-3 the shim holds the credential, so a direct call that skips the connector simply has no token. With option 4 the adapter must check the shim's secret, not just the presence of `X-TOON-Payer`.
- **Where the logic lives.** Turning a payer into an upstream credential is policy. [ADR 0006](https://github.com/toon-protocol/connector/blob/539dd8f6fcf095cae2e61ba17232a97be9642686/docs/adr/0006-the-connector-is-mechanism-not-policy.md) keeps that out of the connector, so it belongs in the shim. #2's citation of [ADR 0047](https://github.com/toon-protocol/connector/blob/539dd8f6fcf095cae2e61ba17232a97be9642686/docs/adr/0047-the-configuration-schema-is-implementation-detail-capabilities-are-law.md) checks out: that ADR exists and makes operator-expressible capabilities law, so even a static route-level upstream-header feature would be a protocol capability, not a config tweak. A per-payer mapping would go further still.

## Which Earning Overlaps survive with only a shared identity

With one shared service token, all payers are one `member` identity. Every row any payer creates, private or org-visible, is readable and writable by every other payer, and so are the dedicated org's other rows.

The safe Actions are those whose result doesn't depend on who calls:
- **Read-only lookups** over data the author meant to sell to anyone.
- **Stateless computations** (transform, generate, analyze an input), where nothing per-payer is stored.
- **Shared-write Actions** whose output is meant to be public to all payers, such as a public board.

Actions that store a payer's own resources need per-payer identity. That includes recordings, forms, documents and anything built on `ownableColumns()`. The `allowedActions` grant on the service token is the tool for restricting the shared identity to the safe set.

## Newly surfaced questions

- **Can the shim map one wallet, not one channel?** Should it resolve `X-TOON-Payer` to the channel counterparty, so a payer keeps its rows across channels? Where would that lookup come from, given ADR 0040 keeps it off the hot path?
- **Which option goes into the prototype?** Option 1 is zero-touch but asks the author for the deployment secret. Option 4 is the honest design but needs app code. That choice decides whether the leading Earning Overlap is a full go or a conditional go.
