# Max-Forwards

## Quick Summary

`Max-Forwards` is a **request** header that limits **how many more times a request may be forwarded by intermediaries** (proxies/gateways) before it must be answered — a hop counter, expressed as a decimal integer (`Max-Forwards: 3`). It applies **only to the `TRACE` and `OPTIONS` methods**. Each proxy that forwards such a request **decrements** the value by one; when a proxy receives the request with `Max-Forwards: 0`, it must **not** forward it further and must instead respond itself. Its purpose is **loop detection and diagnostic hop control**: it lets you probe exactly how far a request travels through a proxy chain and prevents `TRACE`/`OPTIONS` requests from looping infinitely through misconfigured intermediaries. It is a niche, diagnostic header — most application developers never set it — and it's closely tied to the [`TRACE`](../04-Response-Headers/Allow.md) method, which is itself **frequently disabled** for security reasons (Cross-Site Tracing). Understanding `Max-Forwards` matters mainly for **network/proxy debugging** and for knowing why `TRACE` is usually turned off.

## What problem does this header solve?

Two problems specific to `TRACE`/`OPTIONS` in proxy topologies.

**First, infinite loops.** `TRACE` is a diagnostic method that asks a server to echo back the request it received, so you can see how intermediaries modified it. `OPTIONS` (with `*`) probes server/proxy capabilities. In a misconfigured mesh of proxies (A forwards to B, B forwards back to A), such a request could loop forever, consuming resources. `Max-Forwards` bounds this: the counter decrements each hop and forces a response at zero, guaranteeing termination.

**Second, targeted hop diagnosis.** Sometimes you want to know *what a specific proxy in the chain sees or does*, not just the final server. By setting `Max-Forwards` to an exact value, you can make the request stop at a chosen hop and have *that* intermediary respond — e.g. `Max-Forwards: 0` makes the *first* proxy answer, `Max-Forwards: 1` makes the *second* answer, and so on. This lets you "walk" the proxy chain and pinpoint which hop rewrites a header, terminates TLS, or misbehaves — analogous to how `traceroute` uses TTL to map a network path.

`Max-Forwards` is essentially the HTTP-layer equivalent of IP's TTL/hop-limit, scoped to the diagnostic methods where hop-by-hop echoing/probing makes sense.

## Why was it introduced?

`Max-Forwards` was introduced with HTTP/1.1 (RFC 2068, 1997; RFC 2616, 1999) and is specified today in **RFC 9110 §7.6.2 (2022)**. It was designed alongside HTTP's explicit proxy model and the `TRACE` method as the mechanism to make `TRACE`/`OPTIONS` **safe and useful across intermediaries** — bounding loops and enabling per-hop diagnosis. It mirrors the long-standing IP concept of a hop limit (TTL) at the application layer, restricted to the two methods where forwarding a diagnostic request through a chain is meaningful. Its importance has always been niche, and it declined further as **`TRACE` fell out of favor**: the **Cross-Site Tracing (XST)** attack (2003) showed `TRACE` could be abused to exfiltrate cookies/`Authorization` headers via reflected echoing, leading most servers/proxies to **disable `TRACE` entirely** — which in turn made `Max-Forwards` (its primary companion) rarely exercised.

## How does it work?

A client sends `TRACE` or `OPTIONS` with `Max-Forwards: N`. Each forwarding intermediary:
1. checks the value; if `0`, it must **not** forward — it responds itself (for `TRACE`, echoing the received request; for `OPTIONS`, returning its capabilities);
2. otherwise, it **decrements** `Max-Forwards` by 1 and forwards the request to the next hop.

```mermaid
sequenceDiagram
    participant C as Client
    participant P1 as Proxy 1
    participant P2 as Proxy 2
    participant O as Origin
    C->>P1: TRACE / (Max-Forwards: 1)
    Note over P1: >0 → decrement to 0, forward
    P1->>P2: TRACE / (Max-Forwards: 0)
    Note over P2: ==0 → do NOT forward; respond here
    P2-->>P1: 200 OK (echoes the request P2 received)
    P1-->>C: 200 OK
    Note over C: Learned exactly what Proxy 2 saw
```

- **Browser behavior:** Browsers do **not** send `TRACE` (it's a [forbidden method](../02-Core-Concepts/Forbidden-and-Restricted-Headers.md) for `fetch`/XHR, specifically to prevent XST), and thus don't set `Max-Forwards` in normal use. It's a tool for command-line/diagnostic clients.
- **Server behavior:** For `TRACE`/`OPTIONS`, an origin/intermediary must honor `Max-Forwards` (decrement or respond at 0). Ignored for other methods. Most production servers **disable `TRACE`** outright (returning `405`), so `Max-Forwards` often has no effect.
- **Proxy behavior:** This is its home — proxies decrement and forward, or respond at 0. Also used with `OPTIONS *` to probe a specific hop's capabilities.
- **CDN/reverse proxy behavior:** Typically **block `TRACE`** for security; may honor `Max-Forwards` for `OPTIONS`. Nginx doesn't support `TRACE` by default.

## HTTP Request Example

Probing the first proxy (make it respond, don't forward):

```http
TRACE / HTTP/1.1
Host: app.example.com
Max-Forwards: 0
```

Walking to the second hop:

```http
OPTIONS * HTTP/1.1
Host: app.example.com
Max-Forwards: 1
```

## HTTP Response Example

A `TRACE` response echoes the request the responding hop received (`message/http`), revealing intermediary modifications:

```http
HTTP/1.1 200 OK
Content-Type: message/http

TRACE / HTTP/1.1
Host: app.example.com
Max-Forwards: 0
X-Forwarded-For: 203.0.113.7
Via: 1.1 proxy1
```

(The echoed body shows headers *added by intermediaries* — which is exactly why `TRACE` is a security risk and usually disabled.)

A hardened server refusing `TRACE`:

```http
HTTP/1.1 405 Method Not Allowed
Allow: GET, POST, PUT, DELETE, OPTIONS
```

## Express.js Example

Most Express apps should **disable `TRACE`** rather than implement `Max-Forwards`; the header only matters if you deliberately support these diagnostic methods:

```js
const express = require('express');
const app = express();

// 1) RECOMMENDED: reject TRACE outright (defense against Cross-Site Tracing).
app.use((req, res, next) => {
  if (req.method === 'TRACE') {
    return res.status(405).set('Allow', 'GET, POST, PUT, DELETE, OPTIONS').end();
  }
  next();
});

// 2) If you (rarely) act as a gateway honoring Max-Forwards for OPTIONS:
app.options('*', (req, res) => {
  const mf = req.headers['max-forwards'];
  if (mf !== undefined) {
    const n = parseInt(mf, 10);
    if (n === 0) {
      // Must NOT forward; respond with our own capabilities here.
      return res.set('Allow', 'GET, POST, PUT, DELETE, OPTIONS').status(204).end();
    }
    // If we were forwarding upstream, we'd decrement and pass it on:
    // req.headers['max-forwards'] = String(n - 1); proxyToUpstream(req, res);
  }
  res.set('Allow', 'GET, POST, PUT, DELETE, OPTIONS').status(204).end();
});

app.listen(3000);
```

Why each piece matters: for the overwhelming majority of apps, the correct action is route 1 — **disable `TRACE`** (return `405`), which neutralizes XST and makes `Max-Forwards` moot. You only touch `Max-Forwards` (route 2) if your service genuinely acts as a *forwarding gateway* for `OPTIONS`/`TRACE`, in which case you must decrement-and-forward when >0 and respond-without-forwarding at 0. That decrement-or-respond logic is the entire contract; getting it wrong (e.g. forwarding at 0) breaks loop protection.

## Node.js Example

Raw `http` — disable `TRACE`, honor `Max-Forwards` if gatewaying:

```js
const http = require('http');

http.createServer((req, res) => {
  // Disable TRACE (XST防御).
  if (req.method === 'TRACE') {
    res.statusCode = 405;
    res.setHeader('Allow', 'GET, POST, OPTIONS');
    return res.end();
  }

  if (req.method === 'OPTIONS') {
    const mf = req.headers['max-forwards'];
    if (mf !== undefined && parseInt(mf, 10) === 0) {
      // Reached the hop budget → respond here, do not forward.
      res.statusCode = 204;
      res.setHeader('Allow', 'GET, POST, OPTIONS');
      return res.end();
    }
    // (If forwarding: decrement req.headers['max-forwards'] and proxy upstream.)
    res.statusCode = 204;
    res.setHeader('Allow', 'GET, POST, OPTIONS');
    return res.end();
  }

  res.statusCode = 200;
  res.end('ok');
}).listen(3000);
```

The rule: at `Max-Forwards: 0`, respond without forwarding; otherwise decrement before forwarding.

## React Example

React has **zero interaction** with `Max-Forwards`:

1. **Browsers can't send `TRACE`.** The Fetch/XHR spec **forbids** the `TRACE` method (to prevent Cross-Site Tracing), so a React app cannot issue `TRACE` requests, and `Max-Forwards` is only meaningful for `TRACE`/`OPTIONS`.

2. **`OPTIONS` preflights are automatic and browser-controlled.** The [CORS preflight](../07-CORS/CORS-Overview.md) `OPTIONS` is generated by the browser, not your code, and you can't attach `Max-Forwards` to it (it's not something the Fetch API lets you control meaningfully).

3. **It's a network-diagnostic tool, not an app concern.** If you're debugging why a React app's requests are mangled by a proxy, `Max-Forwards`/`TRACE` are tools *you'd use from curl at the command line*, not from React code. And since `TRACE` is usually disabled server-side, even that is often unavailable.

## Browser Lifecycle

There is **no browser lifecycle** for `Max-Forwards`:
1. Browsers don't send `TRACE` (forbidden method).
2. They don't let JS set `Max-Forwards` on the `OPTIONS` preflights they generate.
3. It plays no role in normal page/resource loading.
Its entire life is in **command-line/diagnostic clients and proxy chains**, never in the browser.

## Production Use Cases

- **Proxy-chain diagnosis** (network/ops): walking hops with `Max-Forwards: 0,1,2,...` (via `OPTIONS`, since `TRACE` is usually disabled) to find which intermediary alters a request.
- **Loop protection** for `TRACE`/`OPTIONS` in complex proxy topologies.
- **Capability probing** of a specific hop with `OPTIONS *` + `Max-Forwards`.
- **Understanding why `TRACE` is disabled** (security posture/compliance).
- Otherwise: **rare** — most apps never set or handle it.

## Common Mistakes

- **Leaving `TRACE` enabled.** The real mistake associated with this header — `TRACE` enables Cross-Site Tracing (cookie/`Authorization` exfiltration). Disable it.
- **Forwarding at `Max-Forwards: 0`.** Breaks loop protection; you must respond, not forward.
- **Not decrementing when forwarding.** Also breaks the counter; decrement by exactly 1 per hop.
- **Applying it to methods other than `TRACE`/`OPTIONS`.** It only applies to those two; ignore it elsewhere.
- **Expecting browsers to use it.** They can't send `TRACE`; it's a diagnostic-client concern.
- **Relying on `TRACE` for diagnosis.** It's usually disabled; use `OPTIONS` + `Max-Forwards`, `Via`, or distributed tracing instead.
- **Confusing it with retry/`Max-Forwards`-like app logic.** It's strictly an HTTP hop counter for two methods.

## Security Considerations

- **Cross-Site Tracing (XST) — the headline risk.** `TRACE` echoes the received request, including cookies and `Authorization` headers added by the browser/intermediaries. Combined with an XSS or a way to force a `TRACE`, an attacker could read otherwise-`HttpOnly` credentials. **Disable `TRACE`** on servers, proxies, and CDNs — this is the standard hardening, and it's why `Max-Forwards` is rarely exercised.
- **Information disclosure via `TRACE` echoes.** Even without XST, `TRACE` responses reveal internal headers, proxy identities ([`Via`](../14-Proxies/Via.md)), and infrastructure details. Disabling `TRACE` prevents this leak.
- **`Max-Forwards` itself is benign** but is the control knob for methods (`TRACE`) that are risky — treat the *methods* as the security concern.
- **`OPTIONS` capability probing** can reveal your method surface ([`Allow`](../04-Response-Headers/Allow.md)); expose only what's necessary.
- **Loop-based DoS** that `Max-Forwards` mitigates: ensure forwarding hops honor it (or, more simply, disable `TRACE`).

## Performance Considerations

- **Negligible;** it's a tiny integer on rare diagnostic requests.
- **Loop prevention** is its performance contribution — bounding runaway `TRACE`/`OPTIONS` forwarding.
- **No impact on normal traffic** (doesn't apply to `GET`/`POST`/etc.).
- **Diagnostic value:** correctly walking hops speeds up root-causing proxy issues (an MTTR benefit), though modern distributed tracing usually supersedes it.

## Reverse Proxy Considerations

Nginx: disable `TRACE`, and note it doesn't natively "walk" `Max-Forwards`:

```nginx
server {
  # Block TRACE (and TRACK) to prevent Cross-Site Tracing.
  if ($request_method ~* "^(TRACE|TRACK)$") {
    return 405;
  }

  location / {
    proxy_pass http://app_upstream;
    # Nginx does not implement TRACE; Max-Forwards handling is minimal.
    # Rely on Via / X-Forwarded-* / tracing headers for hop diagnosis instead.
  }
}
```

Key points: the standard posture is to **reject `TRACE`** at the edge. For hop diagnosis, prefer [`Via`](../14-Proxies/Via.md), `X-Forwarded-*`, and distributed-tracing headers (`traceparent`, `x-request-id`) over `TRACE`/`Max-Forwards`, which are largely disabled/limited.

## CDN Considerations

- **CDNs block `TRACE`** by default (XST prevention); `Max-Forwards` therefore rarely does anything for `TRACE` through a CDN.
- **`OPTIONS` handling:** CDNs may answer or forward `OPTIONS` (esp. CORS preflights) and could honor `Max-Forwards`, but this is niche.
- **Prefer vendor trace headers** (`cf-ray`, `x-amz-cf-id`, etc.) and distributed tracing over `TRACE`/`Max-Forwards` for diagnosing CDN hops.
- **Security scanners** flag enabled `TRACE`; ensure it's off end-to-end.

## Cloud Deployment Considerations

- **Load balancers / API gateways** typically disable `TRACE`; confirm as part of hardening.
- **Diagnosis in cloud:** use distributed tracing (X-Ray, Cloud Trace, OpenTelemetry) and forwarding/trace headers rather than `TRACE`/`Max-Forwards`.
- **Compliance:** many baselines explicitly require `TRACE` disabled — which makes `Max-Forwards` a non-factor.
- **Meshes (Envoy/Istio):** provide rich tracing; `Max-Forwards`/`TRACE` are not the tool of choice.

## Debugging

- **curl (probe a hop):** `curl -X OPTIONS -H 'Max-Forwards: 0' https://host/` to make the first hop respond; increment to walk hops. (`TRACE` — `curl -X TRACE` — is usually blocked.)
- **Check TRACE is disabled:** `curl -X TRACE https://host/` should return `405`/blocked; if it echoes your request, `TRACE` is dangerously enabled.
- **Hop mapping alternatives:** use [`Via`](../14-Proxies/Via.md), `X-Forwarded-*`, and `traceparent`/`x-request-id` to understand the chain instead.
- **Security scan:** tools like Nikto/`nmap http-methods` flag enabled `TRACE`.
- **Node/Express:** log `req.method` and `req.headers['max-forwards']` if you implement gateway behavior.

## Best Practices

- [ ] **Disable `TRACE`** (and `TRACK`) on servers, proxies, and CDNs — the key action associated with this header.
- [ ] If acting as a forwarding gateway, **decrement** `Max-Forwards` when >0 and **respond without forwarding** at 0.
- [ ] Only apply `Max-Forwards` to `TRACE`/`OPTIONS`; ignore it for other methods.
- [ ] Prefer [`Via`](../14-Proxies/Via.md), `X-Forwarded-*`, and distributed tracing for hop diagnosis.
- [ ] Don't expect browsers to use it (they can't send `TRACE`).
- [ ] Treat it as a **niche diagnostic** control, not an app feature.
- [ ] Verify `TRACE` is disabled in security scans/compliance checks.

## Related Headers

- [Via](../14-Proxies/Via.md) — records the proxy chain (identities); the preferred modern way to understand hops.
- [Allow](../04-Response-Headers/Allow.md) — lists supported methods; `TRACE` should be excluded, and `OPTIONS` returns it.
- [X-Forwarded-For](../14-Proxies/X-Forwarded-For.md) / [Forwarded](../14-Proxies/Forwarded.md) — forwarding metadata useful for hop diagnosis.
- [Connection](./Connection.md) — connection/hop management (different concern).
- [Forbidden and Restricted Headers](../02-Core-Concepts/Forbidden-and-Restricted-Headers.md) — why browsers can't send `TRACE`.
- [Proxies Overview](../14-Proxies/Proxies-Overview.md) — the intermediary model `Max-Forwards` operates within.

## Decision Tree

```mermaid
flowchart TD
    A[Dealing with TRACE/OPTIONS + Max-Forwards?] --> B{Am I an application server?}
    B -- Yes --> C[Disable TRACE (405).<br/>Ignore Max-Forwards for normal methods.]
    B -- Forwarding gateway --> D{Max-Forwards == 0?}
    D -- Yes --> E[Respond here; do NOT forward]
    D -- No --> F[Decrement by 1, forward upstream]
    A --> G[Need to diagnose proxy hops?]
    G -- Yes --> H[Prefer Via / X-Forwarded-* / tracing<br/>(TRACE usually disabled)]
```

## Mental Model

Think of `Max-Forwards` as the **"deliver-by hand count" you write on a memo being passed down a chain of assistants** — "pass this along at most 2 more times, then whoever's holding it must answer me directly." It's the office-memo version of a **package that self-destructs after N handoffs**, or, more precisely, the HTTP twin of network `traceroute`'s TTL: by setting the count to exactly 0, 1, 2..., you can force *each successive assistant* in the chain to be the one who replies, letting you interrogate them one at a time — "what did *you* receive? what did *you* change?" — to find which desk garbled the memo. Two things keep it honest: an assistant holding a memo marked "0 more handoffs" must **stop and reply**, never pass it on (loop protection), and each one who *does* pass it must **tick the counter down by one**. The catch is that the most powerful version of this interrogation — the `TRACE` memo that says "echo back *exactly* what reached you, including any secret notes stapled on by the mailroom" — turned out to be a security disaster (it could expose secret credentials), so most offices simply **banned that memo type entirely** — which is why, in practice, this clever hop-counting trick is a museum piece you'll rarely get to use.
