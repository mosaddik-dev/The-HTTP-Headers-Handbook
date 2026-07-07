# Header Size Limits

> This is a **concept page**, not a single-header page. It covers the practical, cross-cutting reality that **HTTP headers have size limits** — enforced independently by browsers, servers, proxies, CDNs, and load balancers — and what happens (usually a `431` or `4xx`/`400`/`502`) when you exceed them. Understanding these limits is essential for anything that accumulates header data: big cookies, JWTs in [`Authorization`](../09-Authentication/Authorization.md), long [`Cache-Control`](../06-Caching-Headers/Cache-Control.md)/[`Set-Cookie`](../08-Cookies/Set-Cookie.md) sets, deep [`X-Forwarded-*`](../14-Proxies/X-Forwarded-For.md) chains, or verbose custom headers.

## Quick Summary

There is **no single HTTP header size limit in the spec** — RFC 9110 explicitly declines to mandate one and instead requires servers to support "reasonable" sizes and to respond with **`431 Request Header Fields Too Large`** when a client's headers are too big. In practice, **every hop enforces its own limits**, and they differ: browsers cap what they'll send, Node.js caps incoming headers (default **16 KB** total via `--max-http-header-size`), Nginx uses `large_client_header_buffers` (default **8 KB** per buffer), Apache `LimitRequestFieldSize` (~8 KB per header), and cloud load balancers/CDNs impose their own (often **8–64 KB** total, with per-header sub-limits). Exceeding the limit produces confusing failures — `431`, `400 Bad Request`, `413`, or an opaque `502`/`494` — that are **notoriously hard to debug** because the request often fails at an intermediary before your app sees it. The usual culprits are **oversized cookies** (session bloat, many third-party cookies), **large JWTs** in `Authorization`/`Cookie`, and **accumulated forwarding headers**. The fix is almost always to **shrink what you put in headers** (small session IDs instead of fat tokens, server-side session state, cookie hygiene) rather than to keep raising limits.

## What problem does this concept address?

Headers are parsed into memory *before* a server can even decide what to do with a request, so **unbounded headers are a denial-of-service vector**: a malicious client could send megabytes of headers to exhaust server memory/CPU. Every HTTP implementation therefore caps header size defensively. But those caps create a second, everyday problem: **legitimate applications accumulate header data** — a growing session cookie, a JWT with lots of claims, third-party cookies, a chain of [`X-Forwarded-For`](../14-Proxies/X-Forwarded-For.md)/[`Via`](../14-Proxies/Via.md) entries across proxies — and can silently blow past a limit *somewhere in the request path*, producing failures that are:
- **inconsistent** (works locally, fails behind the CDN/LB, or vice-versa),
- **opaque** (a `502`/`400` with no clear "headers too big" message),
- **user-specific** (only users with large cookies/tokens are affected),
- **intermittent** (fails once the cookie/token crosses the threshold).

This concept page exists so you (a) know the limits exist and roughly what they are per tier, (b) recognize the symptoms, and (c) design to *avoid* the limits (keep headers small) rather than discover them in production.

## Why do these limits exist?

RFC 9110 (2022) deliberately **does not specify a maximum** header or field size — it says a server that receives more than it can process should send **`431 Request Header Fields Too Large`** (introduced in **RFC 6585, 2012**). The reasons are:
- **DoS protection:** headers must be buffered and parsed before request routing, so an attacker sending huge headers could exhaust resources. Bounded buffers prevent this.
- **Implementation practicality:** parsers use fixed or bounded buffers for efficiency; unbounded headers would require unbounded allocation.
- **Historical vulnerabilities:** oversized/malformed headers have enabled crashes and smuggling; strict limits are a hardening measure.

Because the spec leaves the number to implementers, each layer picked its own "reasonable" default — which is exactly why a request can pass one hop and fail another, and why there's no single number to memorize.

## Typical limits by tier

| Layer | Approx. default limit | Config knob |
|---|---|---|
| **Node.js** (http server) | **16 KB** total headers | `--max-http-header-size` (or `maxHeaderSize`) |
| **Nginx** | **8 KB** per buffer (4 buffers) | `large_client_header_buffers`, `client_header_buffer_size` |
| **Apache** | **~8 KB** per header field | `LimitRequestFieldSize`, `LimitRequestFields` (count), `LimitRequestLine` |
| **Browsers (sending)** | varies; often generous but capped (individual cookies ~4 KB) | not user-configurable |
| **AWS ALB** | **~64 KB** total (with per-header limits) | limited config |
| **AWS API Gateway** | **~10 KB** total headers | fixed |
| **Cloudflare** | ~**32 KB** total (~16 KB per header) | plan-dependent |
| **HTTP/2 (HPACK)** | per-stream header list size limits (`SETTINGS_MAX_HEADER_LIST_SIZE`) | server/proxy setting |

These are *approximate and version-dependent* — always verify for your stack. The key takeaway: **the smallest limit in your request path wins**, and it's often an intermediary (CDN/LB/proxy), not your app.

```mermaid
flowchart LR
    A[Browser<br/>sends headers] --> B[CDN<br/>~16-32KB]
    B --> C[Load Balancer<br/>~8-64KB]
    C --> D[Reverse Proxy<br/>Nginx ~8KB/buffer]
    D --> E[App server<br/>Node ~16KB]
    style B fill:#fdd
    style C fill:#fdd
    style D fill:#fdd
    Note[The SMALLEST limit in the path decides success/failure] 
```

## The status codes you'll see

- **`431 Request Header Fields Too Large`** — the *correct* response, but not all servers/intermediaries emit it; some return `400`.
- **`400 Bad Request`** — many servers/proxies use this generically for oversized/malformed headers.
- **`413 Content Too Large`** — technically for *body* size, but sometimes confused; not the header code.
- **`494 Request Header Too Large`** — an Nginx-specific non-standard code sometimes seen internally.
- **`502 Bad Gateway` / opaque errors** — when an intermediary rejects the request before/without a clean status, especially LBs/CDNs.

The diagnostic tell: the failure correlates with **large cookies/tokens** or **many proxies**, affects only some users, and may not appear in your *app* logs (it failed upstream).

## HTTP Examples

A request bloated by cookies + a large JWT (the classic trigger):

```http
GET /dashboard HTTP/1.1
Host: app.example.com
Cookie: session=...; _ga=...; _gid=...; ab_test=...; consent=...; [+ 20 more, total 12 KB]
Authorization: Bearer eyJ...very.long.jwt.with.many.claims... [4 KB]
X-Forwarded-For: 203.0.113.7, 198.51.100.4, 10.0.0.9
```

The server/intermediary rejecting it:

```http
HTTP/1.1 431 Request Header Fields Too Large
Content-Type: text/plain

Request header fields too large
```

## Express / Node.js Example

Node enforces a **total** header limit (default 16 KB); you can raise it, but the better fix is smaller headers:

```js
const http = require('http');
const express = require('express');
const app = express();

// 1) Raising Node's limit (last resort; also settable via `node --max-http-header-size=32768`).
const server = http.createServer({ maxHeaderSize: 32 * 1024 }, app); // 32 KB total headers

// 2) Detect and handle oversized headers gracefully where possible.
//    Node emits 431 automatically when the limit is exceeded — but the request
//    often never reaches Express, so you can't always catch it in a handler.
server.on('clientError', (err, socket) => {
  if (err.code === 'HPE_HEADER_OVERFLOW') {
    socket.end('HTTP/1.1 431 Request Header Fields Too Large\r\n\r\n');
  } else {
    socket.end('HTTP/1.1 400 Bad Request\r\n\r\n');
  }
});

// 3) The REAL fix: keep headers small. Prefer a short session ID over a fat token.
app.use((req, res, next) => {
  const cookieBytes = Buffer.byteLength(req.headers.cookie || '');
  if (cookieBytes > 4096) {
    // Warn early — cookie bloat will eventually break behind a stricter proxy/LB.
    console.warn('Large cookie header', { path: req.path, cookieBytes });
  }
  next();
});

server.listen(3000);
```

Why each piece matters: raising `maxHeaderSize` (step 1) is a *last resort* — it doesn't help if a **stricter intermediary** (Nginx 8 KB, API Gateway 10 KB) is in front of you; the request dies there regardless of your app's limit. That's the crux: **the smallest limit in the path wins**, so raising only your app's limit gives a false sense of safety. The `clientError` handler (step 2) is important because oversized-header requests often **never reach your route handlers** — Node rejects them at the parser, so you must handle them at the socket level. The genuine solution is step 3: **monitor and shrink** header payloads (small opaque session IDs backed by server-side state, cookie cleanup, lean JWTs) so you never approach any tier's limit.

## Node.js Example (raw)

```js
const http = require('http');

// Configure the limit at server creation (or via CLI flag).
const server = http.createServer({ maxHeaderSize: 16 * 1024 }, (req, res) => {
  const totalHeaderBytes = Object.entries(req.headers)
    .reduce((n, [k, v]) => n + Buffer.byteLength(`${k}: ${v}\r\n`), 0);
  res.setHeader('Content-Type', 'application/json');
  res.end(JSON.stringify({ approxHeaderBytes: totalHeaderBytes }));
});

// Reject cleanly when the parser overflows.
server.on('clientError', (err, socket) => {
  const status = err.code === 'HPE_HEADER_OVERFLOW' ? 431 : 400;
  socket.end(`HTTP/1.1 ${status} ${status === 431 ? 'Request Header Fields Too Large' : 'Bad Request'}\r\n\r\n`);
});

server.listen(3000);
```

The point: know your app's `maxHeaderSize`, handle overflow at the socket, and measure real header sizes to catch bloat before it hits a stricter hop.

## React Example

React can't control HTTP header limits, but React apps are a **common source of header bloat** and suffer the failures:

1. **Cookie bloat from client-side libs.** Analytics, A/B testing, consent tools, and auth libraries pile up cookies; since [`Cookie`](../08-Cookies/Cookie.md) is sent on *every* same-site request, a bloated cookie jar can push requests over a proxy/LB limit — manifesting as sudden `400`/`431`/`502` for affected users.

2. **Large tokens in headers.** Storing a big JWT and sending it as `Authorization: Bearer` (or in a cookie) on every request risks the limit. Prefer a **short opaque token / session ID** and keep claims server-side.

```jsx
// Anti-pattern: a huge JWT with many claims sent on every request.
// axios.defaults.headers.common['Authorization'] = `Bearer ${hugeJwt}`; // risky

// Better: a compact token; fetch details from the server as needed.
axios.defaults.headers.common['Authorization'] = `Bearer ${compactToken}`;
```

3. **Symptom recognition:** if some users (those with many cookies / long sessions) get intermittent request failures that clear when cookies are cleared, suspect header size — an infra issue surfacing in your React app.

## Browser Lifecycle

1. The browser assembles request headers (cookies, `Authorization`, custom headers) and enforces its **own** send-side caps (e.g. individual cookies ~4 KB; total varies).
2. It sends the request; the **first intermediary** with a smaller limit that's exceeded **rejects** it (often before your app sees it).
3. Rejection surfaces as `431`/`400`/`502` — the browser shows a failed request; DevTools shows the status.
4. Under HTTP/2/3, header *compression* (HPACK/QPACK) reduces wire bytes, but servers still enforce an **uncompressed** header-list size limit (`SETTINGS_MAX_HEADER_LIST_SIZE`).
5. The user experiences a broken page/API call; clearing cookies often "fixes" it (revealing the cause).

## Production Use Cases (where limits bite)

- **Session/cookie bloat:** large server-side session data stuffed into cookies, or many third-party cookies accumulating.
- **JWT in headers/cookies:** fat tokens with many claims sent on every request.
- **SSO/SAML:** large SAML assertions or auth cookies (a very common `431` cause in enterprise SSO).
- **Deep proxy chains:** accumulated [`X-Forwarded-For`](../14-Proxies/X-Forwarded-For.md)/[`Via`](../14-Proxies/Via.md)/`Forwarded` entries in complex topologies.
- **Verbose custom headers:** tracing/tenancy/debug headers piled onto every request.
- **Migration surprises:** moving behind a stricter CDN/LB suddenly triggers limits that never appeared before.

## Common Mistakes

- **Only raising your app's limit.** A stricter intermediary still rejects the request. Fix the *smallest* limit or (better) the *size*.
- **Putting large state in cookies/tokens.** The root cause of most `431`s. Use short IDs + server-side state.
- **Not monitoring header/cookie size.** Bloat grows silently until it crosses a threshold for some users.
- **Assuming `413` = headers.** `413` is body size; headers are `431` (or `400`).
- **Ignoring per-header vs total limits.** Some tiers cap *each* header (e.g. one giant cookie) *and* the total — you can hit either.
- **Not handling `clientError` in Node.** Oversized requests bypass your routes; handle at the socket.
- **Forgetting HTTP/2 header-list limits.** Compression hides wire size but not the enforced *list* size.
- **Blaming the app** for failures that actually occur at the CDN/LB.

## Security Considerations

- **DoS protection is the reason limits exist.** Don't disable/over-raise them recklessly — huge allowed headers let attackers exhaust memory/CPU during parsing (pre-routing).
- **Request smuggling / parser abuse:** oversized/malformed headers have historically enabled crashes and smuggling; strict, consistent limits across hops are part of the defense (see [Content-Length vs Transfer-Encoding](../10-Compression/Content-Length-vs-Transfer-Encoding.md)).
- **Sensitive data in headers:** large tokens/cookies not only risk limits but also increase exposure surface (logs, caches). Minimize both size and sensitivity.
- **Consistent limits across the path** reduce the risk of hop-to-hop discrepancies that attackers can exploit.
- **`431` info leakage:** keep error responses generic; don't echo the offending headers.

## Performance Considerations

- **Every byte of headers is sent on every request** (especially cookies, which ride along automatically) — bloated headers waste bandwidth and slow requests, particularly on mobile/high-latency links.
- **HTTP/2/3 compress headers** (HPACK/QPACK), greatly reducing the *wire* cost of repeated headers — but the *enforced* limit is on uncompressed size, and parsing cost remains.
- **Smaller headers = faster parsing + lower memory** per request at every hop.
- **Cookie hygiene is a real perf win:** trimming a fat cookie jar reduces per-request overhead across the whole session.
- **Server-side session state** (short cookie ID + lookup) trades a tiny lookup for much smaller requests — usually a net win.

## Reverse Proxy Considerations

Nginx is a frequent place `431`/`400`/`494` originates; tune its buffers if legitimately needed:

```nginx
http {
  # Total large-header buffer capacity: number × size (default 4 × 8k).
  large_client_header_buffers 4 16k;   # raise per-buffer size if you have big cookies/tokens.
  client_header_buffer_size 4k;        # initial buffer for the request line + small headers.

  server {
    location / {
      proxy_pass http://app_upstream;
      # Ensure the upstream (Node) limit is >= Nginx's, or Node rejects what Nginx passed.
    }
  }
}
```

Key points: Nginx returns `400` (or internally `494`) when headers exceed `large_client_header_buffers`. If you raise Nginx's limit, **also raise the app's** (`--max-http-header-size`) so the request doesn't just fail one hop later. Better: reduce header size so you don't need to raise anything.

## CDN Considerations

- **CDNs impose their own header limits** (often the *strictest* hop, e.g. ~16–32 KB total, per-header sub-limits) — a request that works origin-direct may fail through the CDN.
- **Cloudflare/CloudFront/Fastly/Akamai** each publish limits; verify for large-cookie/token apps.
- **Cookie stripping:** some CDNs let you strip/normalize cookies at the edge to reduce header size to origin.
- **Failures at the edge** may not reach origin logs — check CDN analytics/error pages when debugging `4xx`/`5xx` correlated with big headers.

## Cloud Deployment Considerations

- **AWS ALB (~64 KB), API Gateway (~10 KB), CloudFront** each have header limits; API Gateway's is notably small and a common surprise.
- **SSO/SAML on cloud LBs:** large auth cookies frequently hit LB/gateway limits → `431`/`400`.
- **Serverless:** platforms cap request header size (and total request size); large tokens can break invocations.
- **Managed platforms (Vercel/Netlify):** have documented header limits; keep cookies/tokens lean.
- **Service meshes (Envoy):** `max_request_headers_kb` and HTTP/2 header-list settings apply; align across the mesh.

## Debugging

- **curl (measure & reproduce):** `curl -v -H "Cookie: $(printf 'x%.0s' {1..10000})" https://host/` to force an oversized header and see which hop rejects it and with what status.
- **Check DevTools → Network:** a `431`/`400` correlated with a big `Cookie`/`Authorization` is the tell; inspect the request header sizes.
- **Clear cookies test:** if clearing cookies fixes it, header bloat is the cause.
- **Compare paths:** hit the app origin-direct vs through CDN/LB to find the strictest hop.
- **Measure header bytes:** log total header size server-side (as in the Node example) to trend toward limits before they break.
- **Check each tier's logs:** the rejection may only appear in the CDN/LB logs, not your app's.

## Best Practices

- [ ] **Keep headers small** — the primary defense. Prefer short opaque session IDs over fat tokens; keep claims/state server-side.
- [ ] Practice **cookie hygiene**: minimize count/size, scope cookies tightly, clean up stale ones.
- [ ] Know the **smallest limit in your request path** (often a CDN/LB) and design under it.
- [ ] If you raise a limit, raise it **consistently across all hops** (proxy, LB, app) — not just one.
- [ ] Handle oversized requests **gracefully** (Node `clientError` → `431`).
- [ ] **Monitor** header/cookie sizes and alert before you approach limits.
- [ ] Don't over-raise limits (DoS risk); balance capacity against defense.
- [ ] Remember HTTP/2/3 compress the wire but still enforce uncompressed **header-list** limits.
- [ ] Use `431` correctly and keep error responses generic.

## Related Headers / Topics

- [Cookie](../08-Cookies/Cookie.md) / [Set-Cookie](../08-Cookies/Set-Cookie.md) — the #1 source of header bloat; size limits (~4 KB/cookie) and hygiene.
- [Authorization](../09-Authentication/Authorization.md) — large JWTs here are a common `431` trigger.
- [X-Forwarded-For](../14-Proxies/X-Forwarded-For.md) / [Via](../14-Proxies/Via.md) / [Forwarded](../14-Proxies/Forwarded.md) — accumulate per hop, growing header size.
- [Header Syntax and Grammar](./Header-Syntax-and-Grammar.md) — how headers are structured/parsed (buffers).
- [Content-Length vs Transfer-Encoding](../10-Compression/Content-Length-vs-Transfer-Encoding.md) — body framing/limits (distinct from header limits) and smuggling.
- [HTTP Versions and Headers](../01-Introduction/HTTP-Versions-and-Headers.md) — HPACK/QPACK compression and header-list limits in HTTP/2/3.
- [Custom and X- Headers](./Custom-and-X-Headers.md) — verbose custom headers add up.

## Decision Tree

```mermaid
flowchart TD
    A[Getting 431 / 400 / 502 on some requests?] --> B{Correlated with big cookies/tokens?}
    B -- Yes --> C[Shrink headers: short session ID + server-side state,<br/>cookie hygiene, lean JWT]
    B -- Unsure --> D[Measure header size; test with cleared cookies]
    D --> C
    A --> E{Which hop rejects it?}
    E --> F[Test origin-direct vs via CDN/LB to find the strictest limit]
    F --> G{Genuinely need bigger headers?}
    G -- Yes --> H[Raise limit CONSISTENTLY across ALL hops]
    G -- No --> C
```

## Mental Model

Think of header size limits as the **weight-and-size rules for carry-on luggage, enforced separately at every checkpoint on your journey** — the taxi trunk, the curbside desk, the security scanner, the gate, and the overhead bin. There's no single universal rule (the "airline authority" declined to set one), so each checkpoint enforces its own, and **the strictest one you encounter is the one that stops you** — you might sail through the taxi and curbside only to be turned away at the gate. The everyday trouble is *creep*: you keep tucking extra items into your carry-on (cookies, a bulky token, tracing tags) until one day, for travelers with the fullest bags, the gate agent says "too big" (`431`) — and it's baffling because it worked last week and works for lighter travelers. The instinct to demand a bigger bag allowance only helps if *every* checkpoint agrees to it; otherwise you're stopped at the next one. The real fix is almost always to **pack lighter** — carry a claim ticket for your big items (a short session ID pointing to server-side state) instead of hauling everything on your person — so you comfortably clear every checkpoint on every trip.
