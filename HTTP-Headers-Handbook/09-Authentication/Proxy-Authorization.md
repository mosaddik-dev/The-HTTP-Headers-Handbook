# Proxy-Authorization

## Quick Summary

`Proxy-Authorization` is the request header that carries a credential to **an intermediary proxy**, not to the origin server. It is the proxy-tier twin of [Authorization](./Authorization.md): same `<scheme> <credentials>` format (`Basic`, `Digest`, `Negotiate`, `Bearer`), same challenge/response dance — but the challenge status code is **`407 Proxy Authentication Required`** and the server-side challenge header is [Proxy-Authenticate](./Proxy-Authenticate.md) instead of [WWW-Authenticate](./WWW-Authenticate.md). Its single most important property is that it is a **hop-by-hop** header: it authenticates the connection to *the next proxy* and must be consumed and stripped by that proxy, never forwarded to the origin. This clean separation means a request can authenticate to a corporate proxy (`Proxy-Authorization`) *and* to the destination API ([Authorization](./Authorization.md)) simultaneously, with the two credentials riding in different headers so they never collide.

## What problem does this header solve?

Consider a corporate network where every outbound HTTP request must pass through a filtering/logging proxy, and the proxy needs to know *which employee* is making the request (for access control, quota, or audit). The proxy needs a credential — but the request is *also* headed to some external API that has its *own* [Authorization](./Authorization.md) requirement. If both credentials had to share the `Authorization` header, they would clobber each other: the proxy would either strip the origin's credential or forward its own proxy credential to a third party that has no business seeing it.

`Proxy-Authorization` solves this by giving the **proxy hop its own dedicated credential slot**. The client authenticates to the proxy with `Proxy-Authorization`, the proxy validates and *removes* it, and the untouched [Authorization](./Authorization.md) header sails through to the origin. Two authentication relationships — client↔proxy and client↔origin — coexist on one request without interference. It also solves an audit problem: a shared egress proxy can attribute traffic to individual users without the origin ever learning the proxy-side identity.

## Why was it introduced?

`Proxy-Authorization` and the `407` status were introduced in **HTTP/1.1 (RFC 2068, 1997; RFC 2616, 1999)** and now live in **RFC 9110 ("HTTP Semantics", 2022)** alongside the origin-auth framework of RFC 7235. They were added precisely because HTTP/1.0's single [Authorization](./Authorization.md) header could not model the two-party reality of the proxied web: the emerging landscape of corporate firewalls, ISP caching proxies, and gateway devices all needed to authenticate clients *at the proxy boundary* independently of whatever the origin required.

The `Proxy-*` naming and the separate `407` status code (parallel to `401`) were a deliberate, symmetric design: everything you know about `WWW-Authenticate`/`Authorization`/`401` maps one-to-one onto [Proxy-Authenticate](./Proxy-Authenticate.md)/`Proxy-Authorization`/`407`, just for the intermediary hop. Crucially, the spec classified `Proxy-Authorization` as **hop-by-hop** (like `Connection`, `TE`, `Transfer-Encoding`) — it applies only to the single connection between client and the proxy that issued the challenge, and must not be relayed further.

## How does it work?

- **Browser behavior:** When a browser receives `407` + [Proxy-Authenticate](./Proxy-Authenticate.md) from its configured proxy, it shows a **proxy credential dialog** (visually similar to the Basic-auth dialog, but labeled as the *proxy* asking) and then attaches `Proxy-Authorization` to subsequent requests through that proxy — caching the proxy credentials for the session. This is entirely browser-managed; JavaScript cannot set or read `Proxy-Authorization` (it is a **forbidden header name** in the Fetch spec).
- **Server (proxy) behavior:** The proxy reads `Proxy-Authorization`, verifies the credential, and either forwards the request onward (having **stripped the header**) or rejects it with `407`. The origin server behind the proxy never sees `Proxy-Authorization` — if it does, a proxy is misconfigured.
- **Origin behavior:** The origin ignores `Proxy-Authorization` entirely; its concern is the [Authorization](./Authorization.md) header. The two are independent.
- **CDN behavior:** A CDN acting as a reverse proxy in front of an origin normally uses origin-style [WWW-Authenticate](./WWW-Authenticate.md)/`401`, not `407`; `407`/`Proxy-Authorization` is the domain of *forward* proxies (client-side egress). CDNs strip hop-by-hop headers between hops.
- **Reverse proxy behavior:** A reverse proxy *can* demand `Proxy-Authorization`, but conventionally reverse-proxy auth is modeled as origin auth (`401`/[Authorization](./Authorization.md)) because from the client's perspective the reverse proxy *is* the server. `407`/`Proxy-Authorization` is most natural for explicit forward proxies the client is configured to route through.

## HTTP Request Example

A request through a corporate forward proxy that authenticates to *both* the proxy and the origin — note the two distinct headers:

```http
GET http://api.example.com/data HTTP/1.1
Host: api.example.com
Proxy-Authorization: Basic cHJveHl1c2VyOnByb3h5cGFzcw==
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ1c2VyXzEyMyJ9.SIG
Accept: application/json
```

`Proxy-Authorization` is Base64 of `proxyuser:proxypass` (the employee's proxy credential); `Authorization` is the origin API's Bearer token. The proxy consumes and removes the first line before forwarding; only `Authorization` reaches `api.example.com`. Note the **absolute-form request target** (`GET http://api.example.com/data`) — the classic signal that this request is addressed to a forward proxy.

For HTTPS through a proxy, authentication rides the `CONNECT` tunnel setup instead:

```http
CONNECT api.example.com:443 HTTP/1.1
Host: api.example.com:443
Proxy-Authorization: Basic cHJveHl1c2VyOnByb3h5cGFzcw==
```

## HTTP Response Example

If the proxy credential is missing or wrong, the proxy — not the origin — rejects with `407` (see [Proxy-Authenticate](./Proxy-Authenticate.md)):

```http
HTTP/1.1 407 Proxy Authentication Required
Proxy-Authenticate: Basic realm="Corporate Proxy"
Content-Length: 0
```

Once the proxy is satisfied, it forwards the request; the response the client ultimately sees is the origin's (which may itself be a `401` if the *origin* credential is bad — the two failures are independent and use different status codes).

## Express.js Example

Express is normally an **origin/reverse-proxy** server, so it uses `401`/[Authorization](./Authorization.md), not `407`. But you can build an explicit **forward proxy** in Node that demands `Proxy-Authorization` — this is the realistic place the header appears server-side:

```js
const http = require('http');
const net = require('net');
const { URL } = require('url');
const crypto = require('crypto');

const PROXY_USER = 'proxyuser';
const PROXY_PASS = process.env.PROXY_PASS; // never hardcode proxy secrets

// Timing-safe check of the proxy credential (avoid `===`, which leaks length/content).
function proxyCredentialOk(headerValue) {
  if (!headerValue || !headerValue.startsWith('Basic ')) return false;
  const decoded = Buffer.from(headerValue.slice(6), 'base64').toString(); // "user:pass"
  const expected = `${PROXY_USER}:${PROXY_PASS}`;
  const a = Buffer.from(decoded), b = Buffer.from(expected);
  // timingSafeEqual throws on length mismatch, so guard it:
  return a.length === b.length && crypto.timingSafeEqual(a, b);
}

const proxy = http.createServer((clientReq, clientRes) => {
  // 1. Authenticate the PROXY hop. Note we read Proxy-Authorization, NOT Authorization.
  if (!proxyCredentialOk(clientReq.headers['proxy-authorization'])) {
    // 407 (not 401) + Proxy-Authenticate (not WWW-Authenticate) — the proxy is challenging.
    clientRes.writeHead(407, { 'Proxy-Authenticate': 'Basic realm="Corporate Proxy"' });
    return clientRes.end();
  }

  // 2. CRITICAL: strip the hop-by-hop proxy credential before forwarding upstream.
  //    Leaking it to the origin exposes the employee's proxy password to a third party.
  delete clientReq.headers['proxy-authorization'];

  // 3. Forward the request to the real origin; Authorization (origin creds) passes through untouched.
  const target = new URL(clientReq.url); // absolute-form target for forward proxies
  const upstream = http.request({
    host: target.hostname, port: target.port || 80, path: target.pathname + target.search,
    method: clientReq.method, headers: clientReq.headers,
  }, (upRes) => { clientRes.writeHead(upRes.statusCode, upRes.headers); upRes.pipe(clientRes); });
  clientReq.pipe(upstream);
});

// HTTPS: authenticate the CONNECT tunnel, then blindly relay bytes (proxy can't see TLS payload).
proxy.on('connect', (req, clientSocket, head) => {
  if (!proxyCredentialOk(req.headers['proxy-authorization'])) {
    return clientSocket.end('HTTP/1.1 407 Proxy Authentication Required\r\n' +
      'Proxy-Authenticate: Basic realm="Corporate Proxy"\r\n\r\n');
  }
  const [host, port] = req.url.split(':');
  const serverSocket = net.connect(port || 443, host, () => {
    clientSocket.write('HTTP/1.1 200 Connection Established\r\n\r\n'); // tunnel open
    serverSocket.write(head);
    serverSocket.pipe(clientSocket); clientSocket.pipe(serverSocket); // opaque relay
  });
});

proxy.listen(8080);
```

The two load-bearing lines are the `407`/`Proxy-Authenticate` rejection (proxy-tier status/header, *not* `401`/`WWW-Authenticate`) and `delete clientReq.headers['proxy-authorization']` — omit the delete and you forward the proxy secret to every origin the client visits, a serious credential leak. The timing-safe comparison prevents credential brute-forcing via response-time side channels.

## Node.js Example

The example above already uses raw `http`/`net` because that is where forward-proxy logic lives. The **client** side — sending `Proxy-Authorization` through a proxy — looks like this in Node:

```js
const http = require('http');
const token = Buffer.from('proxyuser:proxypass').toString('base64');

// When talking to a proxy, host/port are the PROXY's, and `path` is the ABSOLUTE URL.
const req = http.request({
  host: 'corp-proxy.internal', port: 8080,
  path: 'http://api.example.com/data',                 // absolute-form: tells the proxy where to forward
  method: 'GET',
  headers: {
    'Proxy-Authorization': `Basic ${token}`,           // credential FOR THE PROXY
    'Authorization': `Bearer ${process.env.API_TOKEN}`, // credential FOR THE ORIGIN — separate slot
    Host: 'api.example.com',
  },
}, (res) => { /* on 407 -> fix proxy creds; on 401 -> fix origin creds */ });
req.end();
```

The meaningful contrast with Express: Express applications almost never *issue* `Proxy-Authorization` because they are the destination, not a client behind a proxy. When a Node service *does* call out through a corporate proxy (common in locked-down enterprises), libraries like `https-proxy-agent` build the `CONNECT` + `Proxy-Authorization` handshake for you.

## React Example

React — and browser JavaScript in general — **cannot touch `Proxy-Authorization` at all**. It is on the Fetch spec's **forbidden header names** list, so `fetch(url, { headers: { 'Proxy-Authorization': ... } })` silently drops the header. This is deliberate: proxy authentication is a property of the *user's network configuration*, not the web application. When a browser hits a `407` from the user's configured proxy, the **browser** shows its own proxy-login dialog and manages `Proxy-Authorization` transparently; your React code never sees the `407` (the browser handles it before the response reaches `fetch`), and cannot help with it. The only React-relevant fact: if the user's corporate proxy blocks or mangles requests, your app may see network errors it cannot diagnose or fix from JS — the fix is in the user's proxy settings, not your code.

## Browser Lifecycle

1. **Request routed to the configured proxy** (from OS/browser proxy settings or a PAC file).
2. **Proxy responds `407` + [Proxy-Authenticate](./Proxy-Authenticate.md)** if no valid proxy credential is present.
3. **Browser shows the proxy login dialog** — distinct from the origin's Basic dialog, and labeled as the proxy requesting credentials. JavaScript is not involved and cannot intercept this.
4. **Browser attaches `Proxy-Authorization`** and retries the request through the proxy, then **caches the proxy credentials for the session**, pre-emptively sending them on later requests through the same proxy.
5. **Proxy validates, strips the header, forwards** to the origin (or opens a `CONNECT` tunnel for HTTPS).
6. **Origin auth proceeds independently:** the origin may still issue its own `401`/[WWW-Authenticate](./WWW-Authenticate.md) if the [Authorization](./Authorization.md) header is missing/invalid — a separate handshake the browser or your JS handles separately.

## Production Use Cases

- **Corporate egress proxies:** employees' outbound traffic authenticates to a filtering proxy (Blue Coat, Zscaler, Squid) via `Proxy-Authorization` for per-user access control, DLP, and audit logging.
- **Metered/quota'd proxy services:** commercial forward/residential proxies (data-scraping, geo-testing) authenticate each customer with `Proxy-Authorization` (often Basic) to attribute and bill traffic.
- **Air-gapped/PCI environments:** where servers may only reach the internet through an authenticated proxy, backend services set `Proxy-Authorization` on all outbound calls.
- **CI/build agents behind a proxy:** package managers and Docker pulls in enterprise CI set proxy credentials so build tooling can reach external registries.

## Common Mistakes

- **Forwarding `Proxy-Authorization` to the origin.** It is hop-by-hop; a proxy that relays it leaks the proxy credential to third-party servers. Always strip it after consuming it.
- **Confusing `407` with `401`.** `407` = the *proxy* needs credentials; `401` = the *origin* does. Returning `401` when the proxy is the one challenging (or vice versa) breaks the browser's dialog and the client's retry logic.
- **Using `WWW-Authenticate` on a `407`** (or `Proxy-Authenticate` on a `401`). The status and challenge header must match tiers.
- **Trying to set it from browser JS.** It is a forbidden header; the assignment is silently ignored. Proxy auth is the browser/OS's job.
- **Collapsing proxy and origin credentials into one header.** Defeats the whole point — the two authentication relationships must use their separate headers.
- **Sending Basic proxy credentials to an untrusted/plaintext proxy.** A malicious or unencrypted proxy sees the credential; for HTTPS the `CONNECT` handshake itself is plaintext to the proxy, so the proxy credential is exposed to the proxy (acceptable, it's *for* the proxy) but must never leak beyond it.

## Security Considerations

- **Hop-by-hop discipline is a security control.** The requirement to strip `Proxy-Authorization` before forwarding is what prevents the employee's proxy password from reaching every website they visit. A proxy that fails to strip it is a credential-disclosure vulnerability.
- **Credential exposure to the proxy.** By design the proxy sees the proxy credential in cleartext (Base64 for Basic). This is acceptable — it *is* the intended recipient — but it means the proxy must be trusted, and the channel to it should be TLS (an HTTPS proxy, i.e. proxy connection itself over TLS) where possible. Prefer Digest/Negotiate over Basic for plaintext proxy links.
- **Request smuggling & header injection.** Because `Proxy-Authorization` is hop-by-hop and proxies rewrite it, a proxy chain that inconsistently parses/strips hop-by-hop headers is a classic request-smuggling surface. Keep proxy software patched and consistent.
- **No JavaScript access is protective.** Being a forbidden header, `Proxy-Authorization` cannot be exfiltrated by XSS the way a `localStorage` token can — the browser guards it.
- **Logging.** Proxies must redact `Proxy-Authorization` from access logs exactly as origins redact [Authorization](./Authorization.md).

## Performance Considerations

The header is small; the cost is the **`407` round trip** on the first request through a proxy, mitigated (like `401` for origin auth) by the browser caching proxy credentials and pre-emptively sending them thereafter. For `Negotiate` proxy auth (common with Windows/AD proxies), the handshake is multi-round-trip and **connection-bound**, which interferes with HTTP connection reuse and adds latency — a frequent cause of "why is the corporate network so slow" complaints. For HTTPS the `CONNECT` tunnel is authenticated once at setup, so per-request overhead after the tunnel is established is nil. Chained proxies each add a hop and potentially another `407`.

## Reverse Proxy Considerations

Because a reverse proxy usually *is* the server from the client's view, it conventionally authenticates with `401`/[Authorization](./Authorization.md). If you deliberately front an *upstream* forward proxy that requires `Proxy-Authorization`, Nginx sets it on the connection to that upstream proxy:

```nginx
location / {
    # Talking THROUGH an upstream forward/egress proxy that demands a credential:
    proxy_pass http://upstream-egress-proxy:8080;
    proxy_set_header Proxy-Authorization "Basic cHJveHl1c2VyOnByb3h5cGFzcw=="; # credential for that proxy

    # Ensure hop-by-hop headers from the client are not blindly relayed:
    proxy_set_header Connection "";
}
```

And to make sure a client-supplied `Proxy-Authorization` is not forwarded to your backend (strip it — it's hop-by-hop and none of the backend's business):

```nginx
proxy_set_header Proxy-Authorization "";  # drop any client-sent proxy credential at the boundary
```

The key point mirrors the code example: a reverse proxy must never relay a hop-by-hop proxy credential to the tier behind it.

## CDN Considerations

CDNs are reverse proxies at the edge and use origin-style `401`/[WWW-Authenticate](./WWW-Authenticate.md) for their own auth (signed URLs, edge tokens, Cloudflare Access), not `407`/`Proxy-Authorization`. As RFC-conformant intermediaries they strip hop-by-hop headers between hops, so any `Proxy-Authorization` a client sends is consumed/dropped at the edge and never reaches origin. The practical CDN concern is simply: don't design origin auth around `Proxy-Authorization` expecting it to survive the CDN — it won't.

## Cloud Deployment Considerations

- **NAT gateways / egress proxies:** in locked-down cloud VPCs, outbound traffic often traverses a managed egress proxy (or a self-hosted Squid) that may require `Proxy-Authorization`; set it via `HTTP_PROXY`/`HTTPS_PROXY` env vars, which most SDKs and `undici`/`node-fetch` honor (the proxy agent injects the header).
- **`HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY`:** the standard env-var convention — `http://user:pass@proxy:8080` encodes the `Proxy-Authorization` Basic credential, which the HTTP client library turns into the header. Beware credentials-in-env-var leakage into process listings and logs.
- **Service meshes / sidecars:** Envoy/Istio sidecars typically use mTLS for hop identity rather than `Proxy-Authorization`; the header is a legacy explicit-proxy mechanism, not the mesh pattern.
- **API gateways** authenticate clients with origin semantics (`401`), reserving `407` for true forward-proxy deployments.

## Debugging

- **Chrome DevTools:** you generally *won't* see `Proxy-Authorization` in the Network tab — the browser adds it after the request leaves DevTools' view, and a `407` is handled by the browser's proxy dialog, not surfaced to the page. System proxy logs are the real source of truth.
- **curl:** `curl -x http://proxy:8080 -U proxyuser:proxypass https://api.example.com/data` builds `Proxy-Authorization` for you; `-v` shows the `CONNECT` handshake and the `407`→retry for HTTPS. `--proxy-basic`/`--proxy-digest`/`--proxy-negotiate` pick the scheme.
- **Postman/Bruno:** both have explicit *Proxy* settings separate from request Auth; the proxy credential goes there, and the console/timeline shows the `407` challenge.
- **Node.js:** inspect `req.headers['proxy-authorization']` on your forward proxy; on the client side, log what your proxy agent sends (redacted). Confirm the header is **absent** on the request your origin receives.
- **Squid/Nginx proxy logs:** the proxy's own access log records auth outcome — check there when a `407` loop occurs.

## Best Practices

- [ ] Use `Proxy-Authorization` only for the proxy hop; use [Authorization](./Authorization.md) for the origin — never merge them.
- [ ] Always **strip** `Proxy-Authorization` in the proxy before forwarding upstream (it is hop-by-hop).
- [ ] Challenge with `407` + [Proxy-Authenticate](./Proxy-Authenticate.md); reserve `401` + [WWW-Authenticate](./WWW-Authenticate.md) for origin auth.
- [ ] Prefer Digest/Negotiate over Basic for proxy links that aren't themselves encrypted; encrypt the client↔proxy link when you can.
- [ ] Compare proxy credentials with a timing-safe function; rate-limit to blunt brute force.
- [ ] Redact `Proxy-Authorization` from all proxy access logs.
- [ ] Don't expect the header to survive CDNs/reverse proxies — it's hop-by-hop by design.
- [ ] From backend services, configure proxy credentials via `HTTPS_PROXY` and keep them out of process listings/logs.

## Related Headers

- [Proxy-Authenticate](./Proxy-Authenticate.md) — the `407` challenge this header answers; the proxy-tier counterpart of [WWW-Authenticate](./WWW-Authenticate.md).
- [Authorization](./Authorization.md) — the **origin**-tier credential. Same format, different recipient; the pair coexist on one request precisely because they use separate headers. See its page for the end-to-end vs hop-by-hop distinction in depth.
- [WWW-Authenticate](./WWW-Authenticate.md) — the origin's `401` challenge, mirrored by [Proxy-Authenticate](./Proxy-Authenticate.md).
- [Authentication Overview](./Authentication-Overview.md) — where the `401`/`407` parallel is introduced.

## Decision Tree

```mermaid
flowchart TD
    A[A credential needs to travel with the request] --> B{Who verifies it?}
    B -- The forward proxy the client routes through --> C[Proxy-Authorization<br/>answered to a 407 + Proxy-Authenticate]
    B -- The origin/destination server --> D[Authorization<br/>answered to a 401 + WWW-Authenticate]
    C --> E{Both proxy AND origin<br/>need auth?}
    E -- Yes --> F[Send BOTH headers;<br/>proxy strips Proxy-Authorization,<br/>forwards Authorization]
    E -- No --> G[Send only Proxy-Authorization]
    C --> H{Setting it from browser JS?}
    H -- Yes --> I[Impossible — forbidden header;<br/>the browser manages it]
```

## Mental Model

If [Authorization](./Authorization.md) is the ID you show the doorman at the **building you're visiting**, `Proxy-Authorization` is the ID you show the **security guard at your own office lobby on the way out**. Two different checkpoints, two different guards, two different badges — and you'd never hand your office badge to the outside building (that's the "strip it before forwarding" rule) any more than you'd expect the outside doorman to accept it. The lobby guard (`407`) says "show your office badge before I let you leave the building"; the destination doorman (`401`) says "show your visitor pass before I let you in." You carry both, in separate pockets, and each guard only ever looks at their own. The lobby guard tears up your office badge the moment you're through the door, so it never travels out into the street — which is exactly why the header is hop-by-hop.
