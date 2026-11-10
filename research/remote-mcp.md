# Remote MCP server with per-user OAuth on Firebase

Research for ticket iste2/vestis#6. Researched 2026-11-10 against primary sources (MCP specification and docs, Anthropic and OpenAI connector docs, Firebase and Google Cloud docs). Statements marked "not verified" were not confirmed in a primary source.

## Question

How do we host a remote MCP server for Vestis on Firebase / Google Cloud so that each user connects their own AI client (Claude, ChatGPT, ...) to their own wardrobe with read and write access?

## Short answer

- The current MCP spec is **2026-07-28**. It is stateless: no `initialize` handshake, no `Mcp-Session-Id`, no GET stream, one POST endpoint (Streamable HTTP). 2025-era clients still exist, so the server should serve both eras.
- The MCP server is an OAuth 2.1 **resource server** only. It must publish Protected Resource Metadata (RFC 9728), answer 401 with `WWW-Authenticate`, and validate that each token's audience is the MCP server.
- **Firebase Auth / Identity Platform cannot be the authorization server.** It is only an OIDC relying party. A Firebase ID token also cannot serve as the MCP access token (wrong audience). A bridge is required: either a small authorization server embedded next to the MCP server that uses Firebase Auth for the login step, or a hosted authorization server product.
- Client registration: **Client ID Metadata Documents (CIMD)** are the spec's preferred mechanism; Dynamic Client Registration (DCR) is deprecated but still used as fallback. Claude and ChatGPT both support CIMD and DCR.
- Host on **Cloud Run** (public, in-app auth), TypeScript with the official SDK v2. Cloud Functions 2nd gen is the same platform with less control.
- Claude (all surfaces, all plans) and ChatGPT both support remote MCP with per-user OAuth today.

## 1. MCP authorization spec (current: 2026-07-28)

Versioning: 2026-07-28 is "Current"; protocol versions are dates of the last breaking change ([versioning](https://modelcontextprotocol.io/specification/versioning)). Previous revision: 2025-11-25 ([changelog](https://modelcontextprotocol.io/specification/2026-07-28/changelog.md)).

Roles ([authorization](https://modelcontextprotocol.io/specification/latest/basic/authorization)): the MCP server is an OAuth 2.1 resource server; the client is an OAuth 2.1 client; the authorization server (AS) "may be hosted with the resource server or a separate entity". Authorization is optional in the spec, but a per-user server needs it.

Normative points that shape our design:

- MCP servers MUST implement Protected Resource Metadata (RFC 9728) and the document MUST list at least one entry in `authorization_servers`. Discovery: `WWW-Authenticate: Bearer resource_metadata="..."` on 401, and/or a well-known URI (`/.well-known/oauth-protected-resource[/path]`) ([AS discovery](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/authorization-server-discovery.md)).
- The AS MUST provide RFC 8414 or OpenID Connect discovery metadata; clients try, in order, RFC 8414 with path insertion, OIDC with path insertion, OIDC path appending (same page). The `issuer` in the document must equal the issuer used to build the URL.
- Client registration priority for clients: pre-registered, then CIMD (if AS metadata has `client_id_metadata_document_supported: true`), then DCR (if `registration_endpoint`), else prompt the user ([client registration](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/client-registration.md)). DCR is marked **deprecated** in 2026-07-28 and retained for backward compatibility (changelog, "Deprecated" item 4).
- CIMD: the `client_id` is an HTTPS URL with a path; the AS fetches the JSON document (`client_id`, `client_name`, `redirect_uris` required), MUST verify the `client_id` matches the URL and that the request's `redirect_uri` is in the document (client registration page). SSRF protection is on the AS when it fetches client documents ([security best practices](https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices)).
- PKCE with S256 is required; the AS must advertise `code_challenge_methods_supported` or compliant clients refuse to start ([authorization tutorial](https://modelcontextprotocol.io/docs/tutorials/security/authorization)).
- Clients MUST send the RFC 8707 `resource` parameter (canonical MCP server URL) in authorization and token requests. Servers MUST validate that tokens were issued specifically for them as audience, MUST NOT accept or pass through other tokens, MUST return 401 for invalid or expired tokens, 403 `insufficient_scope` (with `scope`, `resource_metadata`) for scope step-up ([authorization](https://modelcontextprotocol.io/specification/latest/basic/authorization)).
- New in 2026-07-28: the AS SHOULD return `iss` (RFC 9207) and clients MUST validate it if present (SEP-2468); credentials are bound per issuer (changelog minor items 7 to 9).
- Refresh tokens: clients MAY request `offline_access` if the AS lists it; MCP servers SHOULD NOT put `offline_access` in `WWW-Authenticate` scope or `scopes_supported`.
- Security: a proxy-style server that uses a static client ID upstream MUST do per-client consent (confused deputy); consent pages need CSRF protection and anti-framing; redirect URI exact match ([security best practices](https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices)). Scope minimization: list only basic scopes in `scopes_supported` because clients without a challenge scope request all of them.

## 2. Transport

Streamable HTTP in 2026-07-28 ([transport](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http.md)):

- One MCP endpoint accepting POST; each JSON-RPC message is its own POST; server replies with JSON or a request-scoped SSE stream. Required headers include `MCP-Protocol-Version`, `Mcp-Method`, `Mcp-Name`.
- Removed vs 2025-11-25: protocol sessions (`Mcp-Session-Id`), the GET stream, SSE resumability, the `initialize` handshake. Long-lived notifications use `subscriptions/listen`; server-initiated requests (elicitation, sampling) use the multi round-trip pattern.
- Servers MUST validate `Origin` (403 if invalid). HTTP+SSE (2024-11-05) is deprecated.
- Backward compatibility: a server serving only the new era SHOULD answer GET/DELETE with 405 and ignore session headers; clients detect era by trying a modern request first.

Consequence for Vestis: a stateless service fits Cloud Run's scale-out model; no sticky sessions or shared session store needed. Tools that need cross-call state must mint explicit handles bound to the user (state-handle hijacking guidance, security best practices).

## 3. Can Firebase Auth / Identity Platform be the authorization server?

No documented capability:

- Firebase's OIDC support is relying-party only: you configure an external issuer, client ID/secret, and Firebase signs users in through it ([Firebase OIDC](https://firebase.google.com/docs/auth/web/openid-connect)). The page describes nothing about issuing tokens to third-party clients, an authorize/token endpoint, consent, or discovery metadata.
- Firebase ID tokens are meant for your backend and carry `aud` = the Firebase project ID and the securetoken issuer for that project ([verify ID tokens](https://firebase.google.com/docs/auth/admin/verify-id-tokens)). Accepting them as MCP access tokens would violate the spec's audience binding (aud must be the MCP server), and the project-wide `aud` would be shared with every other consumer of the project. Refreshing them needs Firebase's own SDK/API flow, which generic MCP clients do not implement. Treat this as an inference from the cited docs; I did not find a vendor statement either way.

Options for the bridge:

| Option | How | Pros | Cons |
|---|---|---|---|
| A. Embedded minimal AS in the Cloud Run service | `/authorize` shows a login page using the Firebase Auth web SDK (Google/Apple/email as in the app) and an MCP consent page; server verifies the Firebase ID token with the Admin SDK, issues its own single-use PKCE-bound authorization code (stored in Firestore, short TTL), `/token` issues short-lived signed JWT access tokens (`aud` = MCP URL, `sub` = Firebase uid) plus rotating refresh tokens; publishes RFC 8414 metadata and JWKS; fetches CIMD documents (SSRF-hardened); optional DCR | One user directory (Firebase), no per-MAU vendor cost, full control of consent UI and scopes | Security-critical code we own; the MCP tutorial advises using an existing AS product; v2 SDK docs show only resource-server helpers (`mcpAuthMetadataRouter`, `requireBearerAuth`), AS-side helpers not verified |
| B. Hosted AS product (Auth0, Clerk, WorkOS, Keycloak self-hosted) | Product runs authorize/token/CIMD; Firebase would be an upstream login or users are mirrored | Tested OAuth implementation; CIMD support reported by vendors (Auth0, Clerk GA, WorkOS; Keycloak experimental) | Second identity store or federation to Firebase (mapping uid to product subject); per-MAU pricing for a free public app not researched; vendor CIMD claims are from search snippets, not verified at vendor docs |
| C. Firebase ID token as bearer | none | trivial | Violates audience rule; no discovery/registration; refresh unworkable for generic clients; rejected |

`authn0de` (community Firebase-based OAuth2/OIDC server) surfaced in search; it is not an official feature and its maintenance state is unknown. Not recommended.

## 4. Server SDKs

- Official SDK tiers ([SDKs](https://modelcontextprotocol.io/docs/sdk)): TypeScript, Python, C#, Go, Rust, Ruby are Tier 1; Java Tier 2; Swift, PHP, Kotlin Tier 3. SDKs adopt revisions on their own schedule.
- TypeScript SDK v2 (`main` branch) implements 2026-07-28; v1.x gets fixes for at least six months after the v2 release on 2026-07-27 ([repo](https://github.com/modelcontextprotocol/typescript-sdk)). Packages: `@modelcontextprotocol/server`, `@modelcontextprotocol/express`, `@modelcontextprotocol/node`.
- The MCP authorization tutorial gives a working TypeScript resource-server example: `createMcpHandler`, `mcpAuthMetadataRouter` (serves PRM, mirrors AS metadata), `requireBearerAuth` with a JWT verifier (`jose`, JWKS, audience check) ([tutorial](https://modelcontextprotocol.io/docs/tutorials/security/authorization)).
- Legacy clients: `createMcpHandler` defaults to `legacy: 'stateless'`, serving 2025-era clients from the same factory with a fresh instance per request, and answers legacy GET/DELETE with 405; `legacy: 'reject'` makes it modern-only. The v2 server does not serve HTTP+SSE ([legacy clients](https://ts.sdk.modelcontextprotocol.io/v2/serving/legacy-clients)).
- Python (`mcp>=2.2`) and C# SDKs have equivalent token-verifier hooks (tutorial). Dart/Flutter is not an SDK tier; irrelevant here because the Flutter app talks to the backend directly (ADR-0001), not through MCP.

## 5. Hosting: Cloud Run vs Cloud Functions

- Google documents hosting MCP servers on Cloud Run with Streamable HTTP; stdio is unsupported ([Cloud Run MCP](https://docs.cloud.google.com/run/docs/host-mcp-servers)). That page covers IAM-invoker auth for local clients and says nothing about OAuth or public access.
- Public access is required (AI vendors call the server from their cloud): disable the invoker IAM check (`--no-invoker-iam-check`, recommended) or grant `allUsers` `roles/run.invoker`; domain-restricted-sharing org policy blocks the `allUsers` route ([public access](https://docs.cloud.google.com/run/docs/authenticating/public)). All authentication then happens in the app (bearer token validation).
- Request timeout: default 300 s, max 3600 s ([timeout](https://docs.cloud.google.com/run/docs/configuring/request-timeout)). Ample for tool calls; the stateless transport has no long-lived GET stream to keep open.
- Cloud Functions 2nd gen "use Cloud Run" underneath, default concurrency 80, `minInstances` configurable ([2nd gen](https://firebase.google.com/docs/functions/2nd-gen-upgrade)). It can serve the MCP endpoint, but the design needs several routes (MCP, PRM, AS metadata, authorize, consent UI, token, JWKS), which is a normal Express app on a Cloud Run service. Recommendation: Cloud Run service.
- Firebase Hosting rewrites to Cloud Run exist but Hosting has a 60 s request timeout ([Hosting + Cloud Run](https://firebase.google.com/docs/hosting/cloud-run)); use a custom domain pointing at the service (or Hosting only for static pages like the consent UI). Custom-domain mapping options for Cloud Run were not researched here.
- Cold starts matter: Claude gives discovery, registration and token endpoints 10 s and refresh 30 s ([Claude auth docs](https://claude.com/docs/connectors/building/authentication.md)). Consider `min-instances=1` for the auth routes.

## 6. Client support today

Claude ([custom connectors](https://claude.com/docs/connectors/custom/add-unlisted.md), [authentication](https://claude.com/docs/connectors/building/authentication.md)):

- Remote custom connectors by URL on Free (one connector), Pro, Max, Ultra, Team, Enterprise; the same OAuth infrastructure serves claude.ai, Desktop, mobile, Claude Code, Cowork. Transport: Streamable HTTP; a URL ending `/sse` selects legacy SSE.
- OAuth options: CIMD (Anthropic-hosted document, "recommended"), DCR, user-entered pre-registered client. Claude uses CIMD only when AS metadata advertises both `client_id_metadata_document_supported: true` and `none` in `token_endpoint_auth_methods_supported`, else DCR.
- Strictness beyond spec: 401 required to start sign-in (a `WWW-Authenticate` on a 200 is ignored); only the first `authorization_servers` entry is used; PRM `resource` must equal the URL as entered; S256 PKCE; hosted callback `https://claude.ai/api/mcp/auth_callback` (may become claude.com); Claude Code uses loopback redirects on any port (`localhost` and `127.0.0.1`); `/token` must accept `application/x-www-form-urlencoded`, return `invalid_grant` for bad refresh tokens, rotate refresh tokens for public clients; Claude appends `offline_access` when AS lists it; discovery requests come from `160.79.104.0/21`, so a WAF in front of the AS can break it; M2M `client_credentials` unsupported.
- Docs link the 2025-11-25 spec revision for these behaviors; whether Claude already speaks 2026-07-28 was not verified.

ChatGPT ([auth guide](https://developers.openai.com/apps-sdk/build/auth), [connect](https://developers.openai.com/apps-sdk/deploy/connect-chatgpt)):

- Custom MCP server via a public HTTPS Streamable HTTP endpoint (`/mcp`); the connect page does not mention SSE, plans, or mobile.
- OAuth 2.1 with S256 PKCE; PRM and AS metadata required; AS must list S256 or it is unsupported. Registration: CIMD preferred (`none` or `private_key_jwt`), DCR once per connection, or predefined clients. Redirect `https://chatgpt.com/connector_platform_oauth_redirect` if the AS implements RFC 9207 `iss` (must match metadata `issuer` exactly), otherwise a per-connector URL `https://chatgpt.com/connector/oauth/{callback_id}`. Sends `resource`; expects `aud` to match. Optional mTLS client cert and egress IPs for caller verification. No client-credentials, API keys, or service accounts.

Others: the MCP site lists Visual Studio Code, Cursor and others as MCP-capable ([intro](https://modelcontextprotocol.io/clients)) but I did not verify their remote-OAuth support or Gemini. Not needed for v1 as long as the server follows the spec.

## 7. Recommendation

1. **Cloud Run service** (TypeScript, SDK v2, Express, `legacy: 'stateless'`) at a custom domain, public invoker, stateless, `min-instances` 1 on the auth path. Domain logic is not reimplemented: tools call the shared backend (ADR-0001); the MCP layer only maps verified `sub` (Firebase uid) to the backend call. Do not forward the MCP access token downstream (token passthrough is forbidden); use service credentials plus the verified uid.
2. **Authorization server: option A (embedded, Firebase Auth as login) as the default**, because Firebase already holds the users, the app is free and public (per-MAU vendor pricing is a cost the owner would carry), and CIMD plus authorization code with PKCE is a small surface. Keep option B (Auth0/Clerk/WorkOS) as the fallback if the owner prefers not to own security-critical OAuth code. Either way the MCP server stays a pure resource server validating a JWT (`iss`, `aud` = MCP URL, `exp`, scope) with a standard library.
3. **Registration:** support CIMD (advertise `client_id_metadata_document_supported` and `none`), and also expose DCR (rate limited) while older clients and Claude's fallback path need it; plan to drop DCR when it is removed from the spec (deprecated, minimum 12-month window per the lifecycle policy).
4. **Metadata checklist:** PRM `resource` = exact MCP URL; first `authorization_servers` entry = our issuer; `code_challenge_methods_supported: ["S256"]`; `authorization_response_iss_parameter_supported: true` and emit `iss` (ChatGPT stable redirect, spec SEP-2468); `token_endpoint_auth_methods_supported` includes `none`; 401 with `resource_metadata`; avoid WAF/Cloud Armor rules that block vendor egress ranges.
5. **Scopes:** two scopes, e.g. `wardrobe:read` and `wardrobe:write`; put only `wardrobe:read` in `scopes_supported`; challenge `wardrobe:write` through 403 `insufficient_scope` step-up (note: client step-up behavior varies; verify in each client). Never list `offline_access` in the resource's metadata.
6. **Consent:** server-owned consent page naming the client (from CIMD `client_name`), showing redirect URI host, with CSRF protection and anti-framing, stored per user and client.

## 8. Open risks and unknowns

- Security-critical AS code (option A): authorization code storage, refresh rotation, CIMD fetch SSRF, redirect URI matching, consent. Needs a threat-model pass and tests; consider a vetted OAuth library (not researched).
- Whether Claude and ChatGPT already speak 2026-07-28 or still 2025-era; the stateless-legacy default covers both but must be tested against both clients.
- Whether TS SDK v2 offers AS-side helpers; v2 docs reviewed show resource-server helpers only.
- Cloud Run custom-domain mapping and Hosting rewrite constraints (60 s timeout) were not researched in depth; cost of `min-instances` not estimated.
- Hosted AS pricing, CIMD details per vendor, and Firebase-to-AS federation were not verified in vendor docs.
- Step-up (403 `insufficient_scope`) behavior differs per client; the Python SDK doc notes some clients request only previously granted scopes after a 403 (tutorial). Fallback: request both scopes upfront if a client fails step-up.
- Anthropic directory listing (optional later) has extra review requirements not covered here.
- Per-user abuse and cost guardrails for MCP-triggered AI features are a separate map item.
