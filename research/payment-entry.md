# Where can a payment enter an Agent-Native Action call?

Research for [#2](https://github.com/toon-protocol/toon-agent-native/issues/2) (map: [#1](https://github.com/toon-protocol/toon-agent-native/issues/1)). Terms as in [CONTEXT.md](../CONTEXT.md).

Sources are pinned:
- Agent-Native: [`BuilderIO/agent-native@3ab5193`](https://github.com/BuilderIO/agent-native/tree/3ab5193312fd8519675bf19846dec5c298258619) (2026-10-14). Docs are the MDX sources behind agent-native.com/docs (`packages/core/docs/content/`).
- TOON connector: [`toon-protocol/connector@539dd8f`](https://github.com/toon-protocol/connector/tree/539dd8f6fcf095cae2e61ba17232a97be9642686) (2026-10-14) and its ADRs.
- TOON payment proxy RFC: `toon-meta/docs/payment-proxy.md`.

`AN:` below means `https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/`, and `CN:` means `https://github.com/toon-protocol/connector/blob/539dd8f6fcf095cae2e61ba17232a97be9642686/`.

## Answer

**The HTTP API is the one surface where a connector can sit in front of an Action, take payment, and hand the app an ordinary request, with no change to Agent-Native.** It is one POST per Action, it has a path per Action, it returns plain JSON, and there is no session. The app authenticates it with a bearer token, and that is the only problem. The connector forwards whatever headers the payer sent, and it can't add a header of its own. So the bearer has to come from one of three places:

1. **The payer holds an Agent-Native token.** This works today, but then payment isn't the authentication, so it defeats the point.
2. **The Action is public** (`publicAgent.expose`, read-only, `requiresAuth: false`). This also works today, but the app's own URL stays open for free calls too, so the payment only gates the Action if the app can't be reached directly.
3. **A thin TOON-side shim adds the token.** The shim sits between the connector and the app (the route's `handler_url`) and adds `Authorization: Bearer <org service token>`. This is a TOON-side change, not an upstream one, and it is the realistic path. It has one cost: every paying caller acts as the same org-scoped service identity, so all payers can see each other's rows.

**MCP (`/mcp`) can carry a payment, but only as one flat price per HTTP POST.** The transport is stateless, so there's no session to carry. But the Action is named inside the JSON-RPC body, which the connector by law never reads. So the price can't depend on the Action, and `initialize` and `tools/list` cost the same as a `tools/call`. Agent-Native's own `/mcp/tool/:name` route is really the HTTP surface again, and that is the better MCP-adjacent entry point.

**A2A can carry a payment only in a weak form.** A message buys an open-ended LLM loop rather than one Action. Results arrive later, through `tasks/get` polling or an SSE stream. And the authentication is a JWT signed with the app's `A2A_SECRET`, which a proxy could only mint if the app author handed it that secret.

**WebMCP can't carry a payment at all.** It isn't a network surface. It runs inside the user's own browser on their session cookie, which makes it the human-in-the-app's-own-UI case that #1 rules out of scope.

## Per surface

### 1. HTTP API ("Call Actions")

**On the wire.** The call is `POST /_agent-native/actions/<action-id>`. The body is the raw JSON of the action's zod schema, and the response is the bare return value (also JSON). Actions declared with `http: { method: "GET" }` take their arguments as a query string instead. "One request, one JSON body, one JSON response, no handshake, no session, no model in the loop." Errors come back as `{ "error": string }` with status 400, 401, 403, 405 or 500. ([AN:packages/core/docs/content/http-api.mdx](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/docs/content/http-api.mdx), sections "Call an action" and "Errors")

There is a second path to the same Actions: `POST /mcp/tool/:name`, plus the catalog at `GET /.well-known/mcp.json`. It goes through the same dispatcher as `/_agent-native/webmcp/actions/:name`. ([AN:packages/core/docs/content/mcp-protocol.mdx](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/docs/content/mcp-protocol.mdx) "Fetch-first discovery"; [AN:packages/core/src/server/action-routes.ts](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/src/server/action-routes.ts) around line 1766)

**Authentication.** The request needs `Authorization: Bearer <token>`. The token can be:
- a personal token or an org service token minted by `connect`. These are JWTs bound to the audience `{appUrl}/mcp`, and each request checks the token's `jti` against a revocation list.
- a static `ACCESS_TOKEN`/`ACCESS_TOKENS`, for a self-hosted app that has `mcpOwnerEmail` configured.

The HTTP action route reuses the MCP `verifyAuth`, but it passes no owner-email hint. ([AN:packages/core/src/server/auth.ts](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/src/server/auth.ts) `getMcpOAuthBearerSession`, line 1196)

The audience is worked out from the request's `X-Forwarded-Host`/`Host`, or from the app's configured public base URL. So a private upstream address is fine as long as the app knows its public URL. ([AN:packages/core/src/mcp/oauth-route.ts](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/src/mcp/oauth-route.ts) `getMcpOAuthAudiences`)

A server-to-server call with no cookie skips CSRF entirely.

An org service token acts as an implicit org `member`, and the rows it creates are org-scoped. An admin can narrow what it may call with `allowedActions`. ([http-api.mdx](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/docs/content/http-api.mdx) "Service principals")

Public Actions need no token. These are actions with `publicAgent: { expose: true, readOnly: true }`, `requiresAuth: false` and `readOnly: true`. Their `/mcp/tool/:name` paths are registered auth-public. ([action-routes.ts](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/src/server/action-routes.ts) `isPublicWebMcpAction`, line 525; [actions-access-control.mdx](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/docs/content/actions-access-control.mdx) "Expose to public agents")

**Can a connector front it unchanged?** On the transport, yes:
- **One request, one buffered response.** That is exactly what the connector's app client does: it makes the request the envelope describes and returns the complete response. ([CN:crates/connector-runtime/src/app_client.rs](https://github.com/toon-protocol/connector/blob/539dd8f6fcf095cae2e61ba17232a97be9642686/crates/connector-runtime/src/app_client.rs) `HttpAppClient::deliver`)
- **A price per Action.** A price attaches to a handler ([ADR 0020](https://github.com/toon-protocol/connector/blob/539dd8f6fcf095cae2e61ba17232a97be9642686/docs/adr/0020-a-price-is-flat-and-attaches-to-a-handler.md)). The envelope target is confined beneath the handler path ([ADR 0025](https://github.com/toon-protocol/connector/blob/539dd8f6fcf095cae2e61ba17232a97be9642686/docs/adr/0025-an-envelope-target-is-confined-beneath-the-handler-path.md)). So one route per Action, with `handler_url = https://app/_agent-native/actions/<id>`, prices each Action on its own, and the payer can't step outside it.
- **Long Actions.** The packet's own expiry bounds the wait ([ADR 0064](https://github.com/toon-protocol/connector/blob/539dd8f6fcf095cae2e61ba17232a97be9642686/docs/adr/0064-a-deadline-bounds-the-wait-for-an-app-not-the-answer.md)). An Action slower than that deadline is refused `R00` and the payer isn't charged.

**What blocks it: the bearer token.** The connector forwards every header the payer sent, except hop-by-hop headers, `host` and `content-length`. The only headers it adds are `X-TOON-Payer`, `X-TOON-Amount` and `X-TOON-Chain`, and those are attribution only ([app_client.rs](https://github.com/toon-protocol/connector/blob/539dd8f6fcf095cae2e61ba17232a97be9642686/crates/connector-runtime/src/app_client.rs) lines 162-245; [ADR 0040](https://github.com/toon-protocol/connector/blob/539dd8f6fcf095cae2e61ba17232a97be9642686/docs/adr/0040-a-verified-payment-is-stated-to-the-app.md)). A `StaticRoute` has a prefix, a handler URL, a price, a transport policy and a request shape. It has no field for upstream headers ([CN:crates/connector-config/src/route.rs](https://github.com/toon-protocol/connector/blob/539dd8f6fcf095cae2e61ba17232a97be9642686/crates/connector-config/src/route.rs) `StaticRoute`). Agent-Native doesn't know `X-TOON-Payer`.

That leaves the three options from the Answer:

| Option | Agent-Native change | TOON change | Weakness |
|---|---|---|---|
| Payer sends its own bearer | none | none | Payment isn't the authentication. The payer could call the app directly for free. |
| Public Action | the author opts the action in (that's configuration, not an upstream change) | none | Read-only only. The app's public URL serves it unpaid unless the app is network-private. |
| Shim adds the service token | none (the author mints a service token) | a small header-adding proxy at `handler_url`, or a connector route-level upstream-header feature | All payers share one org identity, so rows are org-visible across payers. Still, it gates properly: the token lives only on the TOON side. |

### 2. MCP server ("Expose Your App")

**On the wire.** The endpoint is `POST /mcp`, using MCP Streamable HTTP and JSON-RPC 2.0: `initialize` → `tools/list` → `tools/call`. Each Action is one tool, and there's an extra `ask-agent` meta-tool. A response is either JSON or SSE ([mcp-protocol.mdx](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/docs/content/mcp-protocol.mdx)). The server runs `createMcpHandler(..., { legacy: "stateless", responseMode: "auto" })` and builds a fresh server for every request, so there's no MCP session to pin ([AN:packages/core/src/mcp/server.ts](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/src/mcp/server.ts) line 489). An `ask-agent` call waits inline for at most 25 s, then returns a task handle to poll with `ask_app_status`.

**Authentication.** The client can use any of these:
- remote MCP OAuth 2.1 (discovery, dynamic client registration, PKCE), started by a `401` that carries `WWW-Authenticate`
- a connect-minted JWT
- a static `ACCESS_TOKEN`/`ACCESS_TOKENS`
- an A2A JWT

Scopes are `mcp:read`, `mcp:write` and `mcp:apps` ([mcp-protocol.mdx](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/docs/content/mcp-protocol.mdx) "Authentication"). With a static token, `/mcp` also reads an `X-Agent-Native-Owner-Email` header as the caller's identity, unless `mcpOwnerEmail` is configured, and it grants the full surface ([AN:packages/core/src/mcp/build-server.ts](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/src/mcp/build-server.ts) `matchStaticAccessToken` / `deriveStaticTokenIdentity`). That means a shim could give each payer its own identity on MCP, which the HTTP action route doesn't allow. But it also means the shim gets full-surface power.

**Can a connector front it unchanged?** Mostly. The transport is fine: every message is its own stateless POST. The connector buffers the whole response before replying, so SSE progress is lost but the final result still arrives. **What blocks it:**
- **No price per Action.** The tool name lives in the JSON-RPC body, and the connector never reads the body ([ADR 0067](https://github.com/toon-protocol/connector/blob/539dd8f6fcf095cae2e61ba17232a97be9642686/docs/adr/0067-a-route-declares-its-request-shape-and-the-connector-never-reads-it.md); `payment-proxy.md` "Packet shape"). So `/mcp` gets one flat price for every POST, including `initialize`, `tools/list` and the `ask_app_status` polls.
- **The OAuth discovery dance doesn't fit a paid edge.** The `401` and the `/.well-known/oauth-*` round trips would each be paid packets. A paying agent would have to arrive with a bearer already in hand, or go through the shim.
- **The bearer.** It's the same problem as the HTTP API, and the shim fixes it the same way.

### 3. A2A

**On the wire.** The endpoint is `POST /_agent-native/a2a`, using JSON-RPC 2.0 with `message/send`, `message/stream` (SSE), `tasks/get` and `tasks/cancel`. The agent card is public at `/.well-known/agent-card.json`. A message is text, data or file parts, and it buys a full agent loop, not one Action. On serverless hosts callers must use `async: true` and poll `tasks/get`. ([AN:packages/core/docs/content/a2a-protocol.mdx](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/docs/content/a2a-protocol.mdx))

**Authentication.** The bearer is a JWT signed with `A2A_SECRET`, or with an org-scoped secret looked up by the `org_domain` claim. Its `sub` is the caller's email. There's also an optional legacy static `apiKeyEnv` bearer. Hosted production refuses unauthenticated calls and returns `503` when no secret is configured. ([a2a-protocol.mdx](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/docs/content/a2a-protocol.mdx) "Security", "Auth policy boundary")

**Can a connector front it unchanged?** Only in a weak form. **What blocks it:**
- **The unit of work is an open-ended LLM loop, not an Action.** A flat price per `message/send` has an unbounded cost underneath it.
- **The result is asynchronous.** With `async: true` the paid POST returns only `working`, and the result comes back through later `tasks/get` calls. Each of those is another paid packet at the same flat price, because the connector can't tell the methods apart. `message/stream` is SSE, which the connector buffers.
- **Authentication means handing over a secret.** The shim would need `A2A_SECRET`, or the legacy static key, so it could mint JWTs. That is a stronger credential than a service token.
- **The audience is limited.** In Agent-Native, A2A peers are mostly other Agent-Native apps (the docs call MCP the surface for "external tools calling your app").

### 4. WebMCP

**On the wire.** WebMCP isn't a network protocol. On an authenticated page, `AppProviders` registers the app's backend actions with `document.modelContext` in the browser, and a browser agent calls `executeTool()`. Behind that, the page calls the same dispatcher using the user's own browser session. "WebMCP is not a network MCP server." ([AN:packages/core/docs/content/webmcp.mdx](https://github.com/BuilderIO/agent-native/blob/3ab5193312fd8519675bf19846dec5c298258619/packages/core/docs/content/webmcp.mdx))

**Authentication.** It uses the signed-in user's session cookie, and it runs only while a browser page is open.

**Can a connector front it unchanged?** No. The caller is the human's own browser session inside the app's UI. #1 puts that out of scope ("Humans paying per click in an Agent-Native app's own UI"), and a browser can't speak ILP. The HTTP fallback the WebMCP page describes (`/.well-known/mcp.json` and `/mcp/tool/:name`) is the HTTP surface above.

## Summary table

| Surface | Wire | Auth | Price per Action? | Session / streaming | Works behind a connector with no upstream change? |
|---|---|---|---|---|---|
| HTTP API | `POST /_agent-native/actions/<id>` (or `/mcp/tool/<id>`), JSON in and out | Bearer (connect JWT, service token, static); public Actions need none | **Yes**: one route per Action path | none | **Yes**, if the bearer comes from a TOON-side shim, from the payer, or the Action is public |
| MCP | `POST /mcp`, JSON-RPC, Streamable HTTP | OAuth 2.1, connect JWT, static (+ owner-email header) | **No**: flat per POST, Action is inside the body | stateless; SSE buffered | Yes, at a flat price per message; OAuth discovery doesn't fit |
| A2A | `POST /_agent-native/a2a`, JSON-RPC | JWT signed with `A2A_SECRET` | **No**: buys an agent loop | async tasks with polling; SSE stream | Weakly: needs the app's A2A secret, cost is unbounded, polls are paid |
| WebMCP | `document.modelContext` in the browser | user's session cookie | n/a | browser page | **No**: not a network surface, out of scope |

## Open points raised by this research

- **How should the bearer be added?** The options are a separate header-adding shim at `handler_url`, or a connector feature that adds route-level upstream headers. [ADR 0006](https://github.com/toon-protocol/connector/blob/539dd8f6fcf095cae2e61ba17232a97be9642686/docs/adr/0006-the-connector-is-mechanism-not-policy.md) ("mechanism, not policy") and [ADR 0047](https://github.com/toon-protocol/connector/blob/539dd8f6fcf095cae2e61ba17232a97be9642686/docs/adr/0047-the-configuration-schema-is-implementation-detail-capabilities-are-law.md) bear on which belongs in the connector.
- **One service identity for every payer.** All payers would share an org-scoped identity, so they could read each other's rows on any Action that isn't stateless or read-only. Mapping `X-TOON-Payer` to a per-payer Agent-Native identity works only on `/mcp` (through the owner-email header) and needs full-surface static tokens. It would need checking before any Earning Overlap involves writes.
- **The free direct path.** With the shim, the token never leaves the TOON side, so a call that skips the connector fails with `401`. For public Actions, the app has to be network-private for the payment to gate anything.
