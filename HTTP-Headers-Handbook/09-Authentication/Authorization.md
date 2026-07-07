# Authorization

## Quick Summary

`Authorization` is the request header that carries the caller's credential to the server. It is set by the **client** (a browser after a native login prompt, JavaScript via `fetch`/`axios`, a mobile app, or a service), and it controls whether the server treats the request as coming from an authenticated principal. Its format is fixed by RFC 7235: `Authorization: <scheme> <credentials>`, where the scheme is one of `Basic`, `Bearer`, `Digest`, `Negotiate`, etc. Despite the name, it carries **authentication** material, not an authorization decision. It is the client's half of the `401` challenge/response cycle whose server half is [WWW-Authenticate](./WWW-Authenticate.md). Because it is per-user and secret, it must never be cached, logged, or placed in a URL, and — being a non-safelisted header — it triggers a CORS preflight when set from cross-origin JavaScript.

## What problem does this header solve?

HTTP is stateless: the server remembers nothing between requests. Yet every protected endpoint must re-establish *who is calling* on every single request, without a human re-typing a password each time. `Authorization` is the standardized slot for the compact, machine-verifiable proof of identity that the client attaches to each request. It solves the problem of "how do I prove, on request number 5,000, that I'm the same principal who authenticated at request number 1?" in a way every HTTP client, server, proxy, and framework already understands. Before it, every application invented its own ad-hoc identity parameter; the header gives us one interoperable convention and one associated status code (`401`).

## Why was it introduced?

`Authorization` appeared in HTTP/1.0 (RFC 1945) alongside HTTP Basic authentication and `WWW-Authenticate`, and was formalized in HTTP/1.1's RFC 2617 (Basic and Digest), later split out into **RFC 7235** ("HTTP/1.1: Authentication") in 2014, now consolidated into **RFC 9110**. The era was the early web of protected directories: an admin wanted to password-protect a folder of documents, and the browser needed a standard way to prompt for and transmit those credentials. Basic auth plus a browser dialog driven by `WWW-Authenticate` was the answer. The framework was deliberately *extensible* — a registry of schemes — which is why the modern `Bearer` scheme (RFC 6750, born of OAuth 2.0) slots into the exact same header decades later without any protocol change. The header outlived its original use case because its shape (`scheme` + `credentials`) turned out to be the right abstraction.

## How does it work?

- **Browser behavior:** For `Basic`/`Digest`/`Negotiate`, the browser sets `Authorization` *itself* in response to a `401` + [WWW-Authenticate](./WWW-Authenticate.md): it shows the native credential dialog, then caches the credentials and **re-sends them automatically to the same realm/origin for the rest of the session** — no JavaScript involved. For `Bearer`, the browser does **not** manage the header at all; your JavaScript must attach it explicitly on each `fetch`/`XHR`. The browser never persists or auto-sends a Bearer token. Cross-origin, a script-set `Authorization` header is non-safelisted and forces a **CORS preflight** (`OPTIONS`).
- **Server behavior:** The origin reads `req.headers.authorization`, splits scheme from credentials, verifies them (decode+compare for Basic, signature/expiry check for Bearer JWT, introspection call for opaque tokens), and either populates a request identity (`req.user`) or responds `401` (unknown/invalid credentials) or `403` (valid identity, insufficient permission). The server chooses which schemes it accepts and advertises them in `WWW-Authenticate` on the `401`.
- **Proxy behavior:** `Authorization` is an **end-to-end** header (unlike [Proxy-Authorization](./Proxy-Authorization.md), which is hop-by-hop) — a forward proxy must pass it through untouched to the origin. A proxy authenticating *the proxy hop itself* uses the separate `Proxy-Authorization` header, so the two identities never collide. Proxies should avoid logging `Authorization`.
- **CDN behavior:** CDNs treat the presence of `Authorization` as a strong signal that the response is private and **by default do not cache** responses to requests carrying it (RFC 9111 lets a shared cache store such responses only with an explicit `public`/`s-maxage`/`must-revalidate` directive). Most CDNs also let you include or exclude `Authorization` from the cache key; excluding it while serving protected content is a catastrophic cache-poisoning/leak bug.
- **Reverse proxy behavior:** Nginx/HAProxy/Envoy forward `Authorization` upstream by default. A common pattern is to *terminate* auth at the proxy (validate a JWT via a subrequest, or offload to an auth service) and forward a trusted identity header upstream, stripping or keeping `Authorization` as policy dictates. If the proxy performs its own upstream auth it may *add* an `Authorization` header toward the backend — be careful not to clobber the client's.

## HTTP Request Example

```http
GET /api/orders/42 HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyXzEyMyIsInNjb3BlIjoib3JkZXJzOnJlYWQiLCJleHAiOjE3MzE1MDAwMDB9.SIG
Accept: application/json
```

HTTP Basic (note the value is Base64 of `alice:s3cret`, which is *encoding, not encryption*):

```http
GET /admin HTTP/1.1
Host: internal.example.com
Authorization: Basic YWxpY2U6czNjcmV0
```

## HTTP Response Example

On success, nothing special — the server simply returns the resource. On failure the server issues the challenge (see [WWW-Authenticate](./WWW-Authenticate.md)):

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer realm="api", error="invalid_token", error_description="The access token expired"
Content-Type: application/json

{"error":"invalid_token"}
```

Note the deliberate use of `401` (re-authenticate) rather than `403` (you are known but forbidden). See the [Authentication Overview](./Authentication-Overview.md) for that distinction.

## Express.js Example

A production-shaped JWT Bearer verification middleware using `jsonwebtoken`:

```js
const jwt = require('jsonwebtoken');
const jwksClient = require('jwks-rsa'); // fetches the IdP's public signing keys

// A cached client for the identity provider's JWKS endpoint.
// Verifying RS256 tokens requires the PUBLIC key that matches the token's `kid`.
const jwks = jwksClient({
  jwksUri: 'https://issuer.example.com/.well-known/jwks.json',
  cache: true,            // cache keys in memory so we don't hit the IdP per request
  rateLimit: true,        // cap outbound JWKS requests to survive a key-rotation storm
  jwksRequestsPerMinute: 10,
});

// jsonwebtoken calls this to fetch the key named by the token header's `kid`.
function getKey(header, callback) {
  jwks.getSigningKey(header.kid, (err, key) => {
    if (err) return callback(err);
    callback(null, key.getPublicKey());
  });
}

function requireAuth(req, res, next) {
  const header = req.headers.authorization;      // Node lowercases all header names

  // 1. The credential must be present and use the Bearer scheme. Anything else
  //    is a 401 WITH a challenge so the client knows how to proceed.
  if (!header || !header.startsWith('Bearer ')) {
    res.set('WWW-Authenticate', 'Bearer realm="api"'); // RFC-correct: 401 carries a challenge
    return res.status(401).json({ error: 'missing_token' });
  }

  const token = header.slice('Bearer '.length).trim(); // everything after "Bearer "

  // 2. Verify signature, expiry, and — critically — PIN the algorithm, issuer,
  //    and audience. Pinning algorithms defeats the alg:none and RS256->HS256
  //    key-confusion attacks. Checking `aud` stops a token minted for another
  //    service from being replayed against this one.
  jwt.verify(
    token,
    getKey,
    {
      algorithms: ['RS256'],                 // NEVER trust the token's own alg blindly
      issuer: 'https://issuer.example.com/', // reject tokens from other issuers
      audience: 'https://api.example.com',   // reject tokens meant for another API
      clockTolerance: 5,                     // seconds of skew allowed on exp/nbf
    },
    (err, claims) => {
      if (err) {
        // Distinguish expired (client should refresh) from otherwise invalid.
        const description = err.name === 'TokenExpiredError'
          ? 'The access token expired'
          : 'The access token is invalid';
        res.set(
          'WWW-Authenticate',
          `Bearer realm="api", error="invalid_token", error_description="${description}"`
        );
        return res.status(401).json({ error: 'invalid_token' });
      }

      // 3. AuthN succeeded. Attach the verified identity for downstream handlers.
      //    Everything after this is AUTHORIZATION (a different concern).
      req.user = { id: claims.sub, scope: (claims.scope || '').split(' ') };
      next();
    }
  );
}

// Authorization is separate: identity established, now check permission.
function requireScope(scope) {
  return (req, res, next) => {
    if (!req.user.scope.includes(scope)) {
      // 403, NOT 401: we know who they are; they're simply not allowed.
      return res.status(403).json({ error: 'insufficient_scope' });
    }
    next();
  };
}

app.get('/api/orders/:id', requireAuth, requireScope('orders:read'), (req, res) => {
  // AuthZ that only the business logic can do: ownership check (defeats IDOR).
  const order = db.getOrder(req.params.id);
  if (order.ownerId !== req.user.id) return res.status(403).json({ error: 'forbidden' });
  res.json(order);
});
```

Every guard here earns its place: dropping `algorithms` reopens key-confusion attacks; dropping `audience` lets tokens leak across services; dropping the ownership check is textbook IDOR. For HTTP Basic, the equivalent is `Buffer.from(base64, 'base64').toString().split(':')` followed by a **timing-safe** comparison (`crypto.timingSafeEqual`) against the stored hash — never `===`, which leaks length/content via timing.

### With Passport

```js
const passport = require('passport');
const { Strategy: JwtStrategy, ExtractJwt } = require('passport-jwt');

passport.use(new JwtStrategy(
  {
    // Tells Passport to pull the token from `Authorization: Bearer <token>`.
    jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
    secretOrKeyProvider: (req, rawJwt, done) => getKey(jwt.decode(rawJwt, { complete: true }).header, done),
    algorithms: ['RS256'],
    issuer: 'https://issuer.example.com/',
    audience: 'https://api.example.com',
  },
  (claims, done) => done(null, { id: claims.sub, scope: (claims.scope || '').split(' ') })
));

// `session: false` — Bearer APIs are stateless; do not create a login session.
app.get('/api/me', passport.authenticate('jwt', { session: false }), (req, res) => res.json(req.user));
```

Passport centralizes the extraction/verification but the same pins (`algorithms`, `issuer`, `audience`) remain mandatory — the strategy config is where they live.

## Node.js Example

Raw `http` + `https` client attaching a Bearer token, and a raw server reading it:

```js
const https = require('https');

// Client: attach the credential explicitly. There is no auto-magic for Bearer.
const req = https.request({
  hostname: 'api.example.com',
  path: '/api/orders/42',
  method: 'GET',
  headers: { Authorization: `Bearer ${process.env.ACCESS_TOKEN}` }, // secret from env, never hardcoded
}, (res) => { /* handle response; on 401, refresh and retry once */ });
req.end();
```

```js
// Server: parse the scheme without a framework.
const http = require('http');
http.createServer((req, res) => {
  const auth = req.headers.authorization || '';
  const [scheme, credentials] = auth.split(' ');
  if (scheme !== 'Bearer' || !credentials) {
    res.writeHead(401, { 'WWW-Authenticate': 'Bearer realm="api"' });
    return res.end('unauthorized');
  }
  // ...verify `credentials` (jwt.verify or an introspection call)...
}).listen(3000);
```

The meaningful difference from Express is only ergonomic — Express gives you `req.headers.authorization` and response helpers, but the wire behavior is identical. Note Node normalizes header names to lowercase, so it is always `req.headers.authorization`.

## React Example

React never touches `Authorization` on its own — it's a UI library with no network stack. The header is attached by whatever client you use (`fetch`, `axios`). The two real decisions are *where the token lives* and *how you attach it without scattering it across the codebase*.

```jsx
// A single axios instance is the ONE place the token is attached — do not
// sprinkle `Authorization` across call sites.
import axios from 'axios';

export const api = axios.create({ baseURL: 'https://api.example.com', withCredentials: true });

// Prefer keeping the access token in MEMORY (module scope / React state), not
// localStorage. localStorage is readable by any XSS payload and is the #1 way
// tokens get exfiltrated. Refresh tokens belong in an HttpOnly cookie.
let accessToken = null;
export const setAccessToken = (t) => { accessToken = t; };

api.interceptors.request.use((config) => {
  if (accessToken) config.headers.Authorization = `Bearer ${accessToken}`;
  return config;
});

// Transparent refresh on 401, with single-flight to avoid a refresh stampede
// when many requests 401 at once.
let refreshing = null;
api.interceptors.response.use(
  (r) => r,
  async (error) => {
    const original = error.config;
    if (error.response?.status === 401 && !original._retried) {
      original._retried = true;
      refreshing = refreshing || api.post('/auth/refresh') // refresh token rides an HttpOnly cookie
        .then((r) => { setAccessToken(r.data.accessToken); })
        .finally(() => { refreshing = null; });
      await refreshing;
      original.headers.Authorization = `Bearer ${accessToken}`;
      return api(original); // replay the original request with the fresh token
    }
    return Promise.reject(error);
  }
);
```

The React-specific lesson is architectural: attach the header in *one* interceptor, keep the access token out of `localStorage` (memory + `HttpOnly` refresh cookie is the XSS-resistant pattern), and make refresh single-flight so a burst of `401`s doesn't fire ten parallel refreshes.

## Browser Lifecycle

1. **Bearer (script-driven):** Your code calls `fetch(url, { headers: { Authorization: 'Bearer ...' } })`. If the request is cross-origin, the browser first sends a **preflight `OPTIONS`** (because `Authorization` is not a CORS-safelisted header), waits for `Access-Control-Allow-Headers: Authorization`, and only then sends the real request with the header. The browser never stores or reuses the token — each request re-supplies it.
2. **Basic/Digest (browser-driven):** A `401` + `WWW-Authenticate: Basic realm="x"` triggers the native credential dialog. On submit, the browser sets `Authorization` and retries. It then **caches the credentials for that origin+realm and auto-attaches them to subsequent requests for the session** — which is why there's no clean "logout" for Basic auth (you often have to close the browser or send a bogus `401`).
3. In all cases the header is attached late in request construction, after CORS checks, and is *not* written to the browser cache alongside the response body for private responses.

## Production Use Cases

- **SPA → JSON API:** `Authorization: Bearer <JWT>` attached by an axios/fetch interceptor; the API is stateless and horizontally scalable.
- **Service-to-service:** A backend calls another with `Authorization: Bearer <client-credentials token>` (OAuth 2.0 client-credentials grant) or a scoped API key.
- **Third-party API integration:** Stripe, GitHub, OpenAI, etc. all accept `Authorization: Bearer <api_key>` — the universal convention.
- **Internal tooling / basic-protected dashboards:** `Authorization: Basic` behind TLS and a VPN, where implementation simplicity outweighs UX.
- **OAuth resource servers:** The access token minted by the flow in [Authentication Overview](./Authentication-Overview.md) is presented here on every API call.

## Common Mistakes

- **Believing Basic is encrypted.** `Base64` is reversible in one line. Basic without TLS ships the password in effectively-plaintext. Always TLS.
- **Trusting the JWT's `alg`.** Not pinning `algorithms` server-side enables `alg:none` (accept an unsigned token) and RS256→HS256 confusion (verify an RS256 token using the *public* key as an HMAC secret). Always pass an explicit `algorithms` allowlist.
- **Skipping `aud`/`iss` checks.** A valid token for service B replayed against service A succeeds if you only checked the signature. Verify audience and issuer.
- **Putting the token in the URL.** `?token=...` leaks into access logs, browser history, and the [Referer](../03-Request-Headers/Referer.md) header sent to third parties. Keep credentials in the header.
- **Logging the header.** Request loggers that dump all headers will write live tokens/passwords to disk and your log aggregator. Redact `Authorization` explicitly.
- **`localStorage` tokens.** Any XSS becomes total account takeover because the script can read and exfiltrate the token. Prefer memory + `HttpOnly` cookie.
- **Returning `403` for missing credentials or `401` for permission failures.** Breaks clients' ability to re-auth vs. give up. Use `401` for "who are you," `403` for "not allowed."
- **`===` string comparison of secrets.** Leaks via timing side channel. Use `crypto.timingSafeEqual`.
- **Non-timing-safe Basic compare and no rate limiting** — Basic invites credential-stuffing; rate-limit and lock out.

## Security Considerations

The `Authorization` header *is* the keys to the kingdom, so its threat model is the credential's threat model. **In transit:** TLS is non-negotiable; a plaintext hop exposes every credential. **At rest in the browser:** memory beats `localStorage` beats nothing; use `HttpOnly` cookies for anything long-lived (refresh tokens). **In logs and telemetry:** redact at the logging layer, at the proxy, and in error trackers (Sentry et al. scrub `Authorization` by default — verify it). **Token design:** short-lived access tokens (minutes) limit the blast radius of a leak; pair with rotating, reuse-detecting refresh tokens. **Verification rigor:** pin `alg`, check `iss`/`aud`/`exp`/`nbf`, validate signatures against the correct key (JWKS with `kid`). **Replay:** Bearer tokens are replayable by anyone who holds them — this is inherent; mitigate with short TTLs, TLS, and (for high assurance) sender-constrained tokens (DPoP, mTLS-bound tokens). **CSRF:** the *good* news — an attacker's page cannot set your `Authorization` header cross-origin, so Bearer-in-header is CSRF-immune (unlike cookies). **Brute force:** rate-limit Basic/password endpoints.

## Performance Considerations

The header itself is small, but it has outsized cache impact: its presence marks a response as private, so shared caches (CDN, reverse proxy) will not store it by default — meaning authenticated traffic bypasses the edge cache and hits your origin. That is correct behavior, but it means you can't lean on the CDN for authenticated responses unless you deliberately opt in with `Cache-Control: private`/`public` semantics and careful cache-key design. **Verification cost:** local JWT signature verification is microseconds and needs no network — the whole point of stateless tokens; opaque-token introspection or a session-store lookup adds a round trip per request, so cache introspection results briefly. Cross-origin, the mandatory **preflight** adds a round trip before the first authenticated request — mitigate with `Access-Control-Max-Age` so the browser caches the preflight. JWKS key fetches should be cached in-process (see the Express example) so key rotation doesn't storm the IdP.

## Reverse Proxy Considerations

```nginx
# Forward the client's Authorization upstream (Nginx does this by default, but be
# explicit when you also set other headers, since setting any proxy_set_header
# for a request does not drop the rest — this line documents intent).
proxy_set_header Authorization $http_authorization;

# Offloading auth to the proxy: validate the token via a subrequest to an auth
# service, then forward a trusted identity header and DROP the raw token upstream.
location /api/ {
    auth_request /_authz;                       # subrequest; non-2xx => reject
    auth_request_set $user $upstream_http_x_user;
    proxy_set_header X-User $user;              # trusted identity for the backend
    proxy_set_header Authorization "";          # optional: strip token past the trust boundary
    proxy_pass http://app_upstream;
}
location = /_authz {
    internal;
    proxy_pass http://auth_service;             # returns 200 (+X-User) or 401/403
    proxy_pass_request_body off;
    proxy_set_header Content-Length "";
    proxy_set_header X-Original-URI $request_uri;
}
```

Two rules: (1) never *strip* `Authorization` unless you replace it with an equally trustworthy identity the backend expects; (2) if the backend trusts an `X-User` header, the proxy **must** overwrite/clear any client-supplied `X-User` so a caller can't spoof identity by sending it themselves. Also disable access-log capture of the header (`proxy_hide_header` for responses; log-format scrubbing for requests).

## CDN Considerations

By default (RFC 9111), a shared cache **must not** reuse a stored response for a request bearing `Authorization` unless the response explicitly permits it (`public`, `s-maxage`, `must-revalidate`, or `proxy-revalidate`). This is a safety default that prevents one user's private response from being served to another. Practical guidance:

- **Do not add `Authorization` to the cache key for private content and then cache it** — that's a data-leak/cache-poisoning bug where user A's response is served to user B.
- On Cloudflare/Fastly/CloudFront, authenticated API responses are typically marked "bypass cache." If you *want* to cache a public-but-token-fetched asset, set `Cache-Control: public, max-age=...` deliberately and understand the sharing implications.
- CDNs can strip `Authorization` before forwarding to origin (some default configs do for cached zones) — verify your origin still receives it for dynamic routes.

## Cloud Deployment Considerations

- **AWS API Gateway** has first-class `Authorization`: Lambda authorizers and JWT authorizers parse the header for you and short-circuit unauthenticated requests before they hit your compute. Cognito integrates the same way.
- **ALB/NLB** pass `Authorization` through; ALB can also do OIDC auth at the load balancer and inject `x-amzn-oidc-*` identity headers upstream (which the app must trust only from the ALB).
- **API gateways generally** (Kong, Apigee, Azure API Management, GCP API Gateway) validate JWTs / API keys at the edge and forward a decoded identity — offloading verification from your services. Ensure the gateway strips client-supplied identity headers.
- **Serverless** (Lambda/Cloud Functions/Vercel/Cloudflare Workers): read the header from the event/request object; cache JWKS across warm invocations to avoid per-request key fetches on cold vs warm paths.

## Debugging

- **Chrome DevTools:** Network tab → select the request → *Headers* → *Request Headers* shows `Authorization`. If you see a preceding `OPTIONS` request, that's the CORS preflight the header triggered — check its `Access-Control-Allow-Headers` response includes `Authorization`. Note DevTools may show `Provisional headers are shown` if it can't read them; disable cache and retry.
- **curl:** `curl -v -H "Authorization: Bearer $TOKEN" https://api.example.com/x` (or `-u user:pass` for Basic, which builds the header for you). `-v` prints the exact header sent.
- **Postman:** *Authorization* tab → pick type (Bearer/Basic/OAuth 2.0); Postman constructs the header. Use the *Console* (bottom-left) to see the literal header on the wire.
- **Bruno:** Per-request *Auth* tab; supports Bearer/Basic/OAuth2 and environment variables so tokens aren't committed. The generated header is visible in the *Timeline*.
- **Node.js:** Log `req.rawHeaders` (case-preserving) or `req.headers.authorization` — but **redact before logging in production**. Decode a JWT payload for inspection with `jwt.decode(token)` (decode ≠ verify; never trust decoded claims without `verify`).
- **Express logging:** Configure `morgan`/`pino-http` to *skip or redact* the `Authorization` header. Example with pino: `pino({ redact: ['req.headers.authorization'] })`.

## Best Practices

- [ ] Always over TLS — never send credentials over plaintext HTTP.
- [ ] Bearer tokens: pin `algorithms`, verify `iss`, `aud`, `exp`, `nbf`; use JWKS + `kid` for key rotation.
- [ ] Short-lived access tokens + rotating refresh tokens with reuse detection.
- [ ] Store access tokens in memory; refresh tokens in `HttpOnly`, `Secure`, `SameSite` cookies. Never `localStorage`.
- [ ] Return `401` (+ `WWW-Authenticate`) for missing/invalid credentials; `403` for insufficient permission.
- [ ] Redact `Authorization` in every logger, proxy log, and error tracker.
- [ ] Never put credentials in URLs/query strings.
- [ ] Use `crypto.timingSafeEqual` for secret comparison; rate-limit credential endpoints.
- [ ] Attach the header in ONE place (an interceptor), not scattered across call sites.
- [ ] Set `Access-Control-Max-Age` to amortize the preflight the header triggers.
- [ ] Keep authentication (`req.user`) separate from authorization (ownership/scope checks).

## Related Headers

- [WWW-Authenticate](./WWW-Authenticate.md) — the server's `401` challenge that tells the client which scheme(s) to use in `Authorization`; the two are a matched pair.
- [Proxy-Authorization](./Proxy-Authorization.md) — the same idea but for authenticating to a proxy hop (hop-by-hop), so proxy creds and origin creds don't collide.
- [Cookie](../08-Cookies/Cookie.md) / [Set-Cookie](../08-Cookies/Set-Cookie.md) — the alternative credential-transport mechanism (stateful sessions) with different CSRF/XSS tradeoffs.
- [Cache-Control](../06-Caching-Headers/Cache-Control.md) — governs whether an `Authorization`-bearing response may be cached by shared caches.
- [Referer](../03-Request-Headers/Referer.md) — why you must never leak credentials into URLs.

## Decision Tree

- Building a browser SPA / mobile app talking to a JSON API? → `Authorization: Bearer <token>`, token in memory, refresh in `HttpOnly` cookie.
- Traditional server-rendered app with server-side sessions? → Skip `Authorization`; use [Cookie](../08-Cookies/Cookie.md)/[Set-Cookie](../08-Cookies/Set-Cookie.md) sessions.
- Service-to-service or third-party API? → `Authorization: Bearer <api-key/client-credentials token>`.
- Internal tool behind TLS+VPN, simplicity paramount? → `Authorization: Basic` is acceptable.
- Need instant revocation and centralized control? → Opaque tokens + introspection, or sessions — not raw stateless JWTs.
- Cross-origin from a browser? → Expect and optimize the CORS preflight (`Access-Control-Max-Age`).

## Mental Model

**`Authorization` is the ID you present at the checkpoint, and the scheme name is the *type* of ID.** "Basic" is handing over your actual password on a slip of paper (so only do it inside a sealed tunnel — TLS). "Bearer" is showing a wristband: the guard doesn't check it's *yours*, only that it's valid and unexpired, so never drop it and let it expire fast. The guard (server) validates the ID against records it already trusts — a signature it can check with a public key (JWT, no phone call needed) or a ticket number it looks up in a ledger (opaque token, one phone call). Presenting no ID gets you a `401` with a posted list of "IDs we accept" ([WWW-Authenticate](./WWW-Authenticate.md)); presenting valid ID for a door you're not cleared for gets you a `403`. And the cardinal rule of the checkpoint: the ID is worthless the instant a copy leaks — so it travels only through the sealed tunnel, is never written in the logbook, and is never taped to the outside of the envelope (the URL).
