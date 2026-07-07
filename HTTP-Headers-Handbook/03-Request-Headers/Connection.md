# Connection

## Quick Summary

`Connection` is a **hop-by-hop** control header that governs the *single TCP connection between two adjacent nodes* — it is not addressed to the origin server, it is addressed to whoever is on the other end of *this socket*. It carries two kinds of information: (1) connection-management directives, chiefly `keep-alive` (reuse this socket for more requests) and `close` (I'm done, tear it down after this message); and (2) a **list of other header field names** that are to be treated as hop-by-hop *for this message* and stripped before forwarding. It is set by clients, servers, and every intermediary independently, because each hop manages its own connection. In HTTP/2 and HTTP/3 the header is **forbidden** — the protocols manage connections through their own framing layer.

## What problem does this header solve?

Two problems, one historical and one structural.

**The historical problem: TCP connection reuse.** In HTTP/1.0 the default was one TCP connection per request/response, then close. Opening a TCP connection costs a three-way handshake (one RTT), and with TLS another one-to-two RTTs. For a page that pulls 40 subresources, paying a full connection setup 40 times is catastrophic for latency. `Connection: keep-alive` was the opt-in mechanism to say "don't close this socket, I have more requests coming," turning N handshakes into 1.

**The structural problem: some headers only make sense between two adjacent nodes.** A request may traverse browser → CDN → reverse proxy → app, over four separate connections. A directive like "keep this socket open" is meaningless to forward — the next hop is a *different* socket. HTTP needed a way to mark headers as "consume here, do not forward." `Connection` is both an example of such a header (it is itself hop-by-hop) and the **mechanism** for declaring others hop-by-hop dynamically.

See [End-to-End vs Hop-by-Hop Headers](../01-Introduction/End-to-End-vs-Hop-by-Hop-Headers.md) for the full taxonomy.

## Why was it introduced?

HTTP/1.0 (RFC 1945) had no standard persistent-connection mechanism; an unofficial `Connection: keep-alive` convention emerged (originally a Netscape extension) but it was fragile because intermediaries that didn't understand it would blindly forward it, causing a proxy to hold a connection open that the next hop expected to close — a deadlock.

HTTP/1.1 (RFC 2068, then RFC 2616, now **RFC 9112**) fixed this by (a) making **persistent connections the default** — you no longer ask for keep-alive, you ask to `close` — and (b) formally defining `Connection` as a hop-by-hop header whose tokens are consumed at each hop and never forwarded. This closed the "blind forwarding" hole: a compliant proxy strips `Connection` and re-decides connection management for its own upstream socket.

HTTP/2 (RFC 7540/9113) and HTTP/3 (RFC 9114) then removed the need for it entirely: multiplexing and stream framing make connection management a property of the protocol, so `Connection` (and the headers it typically names) became illegal to send.

## How does it work?

- **Browser behavior:** Browsers manage a connection pool per origin (typically up to 6 concurrent HTTP/1.1 connections per host). On HTTP/1.1 they keep sockets alive and reuse them silently; they generally do **not** expose or let you set `Connection` from `fetch`/`XHR` — it is a [forbidden header name](https://developer.mozilla.org/en-US/docs/Glossary/Forbidden_header_name) that the browser controls. On HTTP/2+ the browser opens one connection per origin and multiplexes; `Connection` is never sent.
- **Server behavior:** The server reads `Connection` to decide whether to keep the socket open (`keep-alive`) or close it (`close`) after writing the response, and echoes its own decision in the response's `Connection` header. It also honors the hop-by-hop *list* semantics: any header named in `Connection` is treated as local to this connection.
- **Proxy behavior:** A forward proxy **terminates** `Connection`: it reads and acts on it, strips it and every header it names, then opens/reuses its *own* connection to the next hop and sets a fresh `Connection` for that hop. It never forwards the client's `Connection` verbatim.
- **CDN behavior:** Identical to a proxy — the CDN edge terminates the client connection (usually HTTP/2 or HTTP/3 client-side) and maintains a separate, often long-lived keep-alive pool to the origin. The client's connection semantics and the origin's are independent.
- **Reverse proxy behavior:** Same termination model. Critically, Nginx defaults to HTTP/1.0 *toward upstreams* with `Connection: close`, so to get keep-alive to your Node app you must opt in explicitly (see [Reverse Proxy Considerations](#reverse-proxy-considerations)). And because `Connection` is hop-by-hop, a WebSocket `Connection: Upgrade` token is dropped unless you re-inject it.

## HTTP Request Example

HTTP/1.1 request explicitly asking to close the socket after the response, and declaring a custom header hop-by-hop:

```http
GET /api/orders HTTP/1.1
Host: api.example.com
Connection: close, X-Edge-Trace
X-Edge-Trace: edge-node-7
Accept: application/json
```

Here `close` tells the server to terminate the connection after responding; `X-Edge-Trace` is declared hop-by-hop, so a compliant intermediary consumes it and does **not** forward it upstream.

A keep-alive request (rarely written by hand on HTTP/1.1 since it's the default, but explicit for illustration):

```http
GET /api/orders HTTP/1.1
Host: api.example.com
Connection: keep-alive
Keep-Alive: timeout=5, max=100
```

## HTTP Response Example

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 42
Connection: keep-alive
Keep-Alive: timeout=5, max=100

{"orders":[],"nextCursor":null}
```

The server agrees to keep the socket open for up to 5 idle seconds or 100 total requests. If it wanted to close instead:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 42
Connection: close

{"orders":[],"nextCursor":null}
```

After sending `Connection: close`, the server MUST close the socket after the body, and the client MUST NOT send more requests on it.

## Express.js Example

You rarely set `Connection` in Express — Node's HTTP server manages it — but you must understand how to *influence* it and when to force a close. Below is a production-shaped example.

```js
const express = require('express');
const app = express();

// 1. The underlying Node server owns keep-alive. Tune it here, not per-response.
const server = app.listen(3000);

// keepAliveTimeout: how long an idle socket stays open waiting for the next
// request. If this is SHORTER than your load balancer's idle timeout, the LB
// can send a request onto a socket Node just closed -> 502s. Set it HIGHER
// than the LB/ELB idle timeout (AWS ALB default is 60s).
server.keepAliveTimeout = 65_000;

// headersTimeout must be >= keepAliveTimeout, or Node may close a reused socket
// mid-request-line. This is a well-known source of intermittent 502s behind ALBs.
server.headersTimeout = 66_000;

// 2. A route that must NOT reuse the connection (e.g., after issuing a response
// whose framing you can't guarantee, or during graceful shutdown).
app.get('/health', (req, res) => {
  // During shutdown we ask clients to stop reusing this socket so the pool drains.
  if (app.locals.shuttingDown) {
    res.set('Connection', 'close'); // hop-by-hop: tells THIS client to not reuse
    return res.status(503).json({ status: 'draining' });
  }
  res.json({ status: 'ok' });
});

// 3. Graceful shutdown: flip the flag, stop accepting, let in-flight finish.
process.on('SIGTERM', () => {
  app.locals.shuttingDown = true;
  server.close(() => process.exit(0)); // stops accepting, waits for idle sockets
});
```

Line-by-line rationale:

- `server.keepAliveTimeout = 65_000` — the single most important production setting related to `Connection`. Node's default (5s) is *shorter* than most cloud load balancer idle timeouts, which creates a race: the LB believes a pooled socket is alive and dispatches a request onto it at the exact moment Node closes it, producing sporadic 502s that are maddening to reproduce. Setting Node's timeout **above** the LB's eliminates the race because Node never closes first.
- `server.headersTimeout = 66_000` — must be `>= keepAliveTimeout`. If it's lower, Node can abort a request whose headers are still arriving on a reused connection. The `+1000` is a defensive margin.
- `res.set('Connection', 'close')` in `/health` — Express/Node will honor this and close the socket after the response. Used here as a *connection-draining* signal during shutdown so pooled clients rebuild against healthy instances. Removing it means clients keep hammering a draining node.
- The `SIGTERM` handler — `server.close()` stops accepting new connections and fires its callback once all keep-alive sockets go idle and close; combined with the `Connection: close` hint, the pool drains cleanly instead of dropping in-flight requests.

## Node.js Example

Raw `http` module, showing both the server's response header and, importantly, the **client** side where keep-alive is an Agent concern, not a header you write:

```js
const http = require('http');

// --- Server: decide connection management per request ---
const server = http.createServer((req, res) => {
  // req.headers.connection is lowercased by Node. On HTTP/1.1 it's usually
  // absent (keep-alive is implied) or 'keep-alive'; 'close' means last request.
  const wantsClose = (req.headers.connection || '').toLowerCase() === 'close';

  res.writeHead(200, {
    'Content-Type': 'text/plain',
    // Node sets Connection automatically to match keep-alive state, but you can
    // override to force a close (e.g., you detected a poisoned socket).
    ...(wantsClose ? { Connection: 'close' } : {}),
  });
  res.end('ok');
});

server.keepAliveTimeout = 65_000;
server.headersTimeout = 66_000;
server.listen(3000);

// --- Client: keep-alive lives in the Agent, NOT in a header you set ---
const agent = new http.Agent({
  keepAlive: true,      // reuse sockets; Node manages the Connection header for you
  maxSockets: 50,       // per-host concurrency cap
  maxFreeSockets: 10,   // idle sockets retained in the pool
  timeout: 60_000,      // socket inactivity timeout
});

http.get({ host: 'api.example.com', path: '/orders', agent }, (res) => {
  res.resume(); // drain the body so the socket returns to the pool for reuse
});
```

Key internals to internalize:

- **On the client you do not set `Connection: keep-alive` yourself.** You configure an `http.Agent` with `keepAlive: true`, and Node emits the correct header and manages the socket pool. Manually setting the header without a keep-alive agent does nothing useful.
- `res.resume()` — a subtle but critical line. A keep-alive socket cannot be returned to the pool until the response body is fully consumed. If you ignore the body, the socket is stuck and you leak connections until `maxSockets` is hit and requests queue forever. Draining (or reading) the body is what frees the socket.
- Node lowercases all incoming header names, so you always read `req.headers.connection`, never `Connection`.

## React Example

React never touches `Connection` directly — it is a [forbidden header name](https://developer.mozilla.org/en-US/docs/Glossary/Forbidden_header_name) in the Fetch spec, so `fetch(url, { headers: { Connection: 'close' } })` is silently ignored by the browser. The browser owns connection management entirely.

What a React engineer *should* understand instead:

```js
// This header is IGNORED by the browser — it controls the socket, not your app.
fetch('/api/orders', { headers: { Connection: 'close' } }); // no effect

// What actually matters for connection reuse in the browser:
// - On HTTP/2 (default for HTTPS on modern CDNs), ONE connection is multiplexed
//   across all your requests to an origin. Batching many small requests is cheap.
// - On HTTP/1.1, the browser caps ~6 connections per origin. If you fire 30
//   requests, 24 queue behind the first 6 -> head-of-line blocking at the app layer.
```

The practical React lesson: connection behavior is dictated by the **protocol negotiated with your CDN/origin**, not by anything in your code. If your API is HTTP/1.1 and your dashboard fires 30 parallel requests, you are queuing behind the 6-connection limit — the fix is HTTP/2 (server/CDN config) or request coalescing, never a `Connection` header.

## Browser Lifecycle

1. **Origin lookup:** For a new request the browser checks its connection pool for a reusable, idle socket to that origin (matching scheme+host+port, and for HTTP/2 the ALPN protocol and certificate).
2. **Protocol decision:** During TLS the browser negotiates ALPN. If `h2` is chosen, connection management is HTTP/2's job — no `Connection` header will ever be sent.
3. **HTTP/1.1 path:** If a live socket exists, reuse it (implicit keep-alive). If not, open a new TCP+TLS connection. The browser does not emit `Connection: keep-alive` explicitly in modern builds because it's the default.
4. **Response handling:** If the response carries `Connection: close`, the browser marks the socket for teardown after the body and removes it from the pool. Otherwise it returns the idle socket to the pool for reuse, subject to `Keep-Alive` timeout hints and the browser's own idle limits.
5. **Pool eviction:** Idle sockets are eventually closed on a timer, on memory pressure, or when the tab/origin is torn down.

## Production Use Cases

- **Graceful rolling deploys / connection draining:** Emitting `Connection: close` from a node that is shutting down forces clients and load balancers to rebuild connections against healthy instances, so no in-flight request lands on a dying process.
- **Load-balancer 502 elimination:** Correctly ordering `keepAliveTimeout > LB idle timeout` (and `headersTimeout >= keepAliveTimeout`) is the canonical fix for intermittent 502s behind AWS ALB/ELB, GCP LB, and Azure.
- **Upstream keep-alive pools:** Reverse proxies and CDNs keep long-lived keep-alive connections to origins to amortize handshake cost across millions of end-user requests. Configuring this pool (`keepalive` in Nginx upstreams) is a major throughput lever.
- **WebSocket upgrades:** `Connection: Upgrade` is the pairing token for the [Upgrade](./Upgrade.md) handshake; managing it correctly through proxies is a core operational task.

## Common Mistakes

- **`keepAliveTimeout` lower than the LB idle timeout.** The #1 cause of ghost 502s behind ALBs. Node closes a socket the LB still thinks is usable.
- **Setting `Connection` from browser `fetch`/`XHR`.** It's a forbidden header; the browser ignores it. Engineers waste hours "fixing" connection behavior this way.
- **Forwarding `Connection` verbatim through a custom proxy.** A hand-rolled proxy that copies all client headers to the upstream leaks a hop-by-hop header, confusing upstream connection management and potentially enabling request smuggling.
- **Not draining response bodies on a keep-alive client agent.** Leaks pooled sockets until the pool is exhausted and requests hang.
- **Expecting `Connection` to reach your app in HTTP/2.** It's stripped/forbidden; code that branches on `req.headers.connection` behaves differently across protocol versions.
- **Listing a header you actually want to survive in `Connection`.** `Connection: close, Authorization` would tell proxies to strip `Authorization`.

## Security Considerations

- **HTTP Request Smuggling:** Mishandling of `Connection` (and the headers it names, especially `Transfer-Encoding`) across a proxy/origin pair with divergent parsing is the root of request smuggling. A defense-in-depth rule: intermediaries must strip hop-by-hop headers and never trust a client-supplied `Connection` list to override protocol behavior. See [Transfer-Encoding](../10-Compression/Transfer-Encoding.md).
- **Header leakage via the `Connection` list:** Because `Connection` can name arbitrary headers as hop-by-hop, an attacker who can inject a `Connection: X-Auth-User` header could strip an internal trust header between proxy and origin. Origins must not rely on client-controllable framing to carry security context; set trust headers at the proxy and strip inbound copies.
- **HTTP/2 downgrade confusion:** A translation layer that maps H2 → H1.1 must synthesize connection management correctly; a mismatch here is a classic smuggling vector (CVE-class "H2.TE"/"H2.CL" attacks).
- **Denial of service via connection hoarding:** Keep-alive lets a client hold sockets open. Without `maxSockets`/connection limits and idle timeouts, a slow-loris-style client can exhaust the server's socket table.

## Performance Considerations

- **Keep-alive is the single biggest HTTP/1.1 latency win:** it removes a TCP handshake RTT (and TLS RTTs) per subsequent request. Never disable it globally.
- **Idle sockets cost memory/FDs:** each pooled socket consumes a file descriptor and kernel buffers. Tune `maxFreeSockets`/`keepAliveTimeout` to balance reuse against resource exhaustion under high fan-out.
- **HTTP/2 obsoletes most of this at the client edge:** one multiplexed connection removes the 6-connection cap and head-of-line blocking at the connection layer (though TCP-level HOL blocking remains until HTTP/3/QUIC).
- **Upstream keep-alive to origin** dramatically raises effective throughput at CDNs/proxies by amortizing handshakes over huge request volumes; the `keepalive` upstream directive is often worth double-digit percentage latency improvements.

## Reverse Proxy Considerations

Nginx's defaults toward upstreams are the gotcha. By default Nginx talks **HTTP/1.0** to upstreams and sends `Connection: close`, so every proxied request opens a fresh socket to your Node app.

```nginx
upstream app_backend {
    server 127.0.0.1:3000;
    keepalive 32;              # keep up to 32 idle keep-alive conns to the upstream
    keepalive_timeout 60s;     # how long an idle upstream conn is retained
    keepalive_requests 1000;   # max requests per upstream conn before recycling
}

server {
    location /api/ {
        proxy_pass http://app_backend;

        proxy_http_version 1.1;        # REQUIRED: HTTP/1.0 has no persistent conns
        proxy_set_header Connection "";# REQUIRED: clear the default "close" so the
                                       # upstream socket is kept alive & reused
    }

    # WebSocket location: here Connection must be "upgrade", not empty
    location /ws/ {
        proxy_pass http://app_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";  # re-inject the hop-by-hop token
        proxy_read_timeout 3600s;
    }
}
```

Why each line exists:

- `keepalive 32` in the `upstream` block — enables the idle connection pool to the backend. Without it, `proxy_http_version 1.1` alone still won't reuse connections.
- `proxy_http_version 1.1` — HTTP/1.0 has no standard persistent connections, so keep-alive to the upstream is impossible without this.
- `proxy_set_header Connection ""` — Nginx's default upstream `Connection` value is effectively `close`; setting it to empty (which Nginx translates to keep-alive when `keepalive` is configured) is what actually enables reuse. **This is the counterintuitive part:** you set `Connection` to an empty string to get keep-alive.
- For WebSockets you do the opposite: set `Connection "upgrade"` and forward `Upgrade`, because those hop-by-hop headers are otherwise dropped. See [Upgrade](./Upgrade.md).

## CDN Considerations

- CDNs terminate the client connection at the edge (commonly HTTP/2 or HTTP/3 client-side) and maintain their **own** keep-alive pool to your origin. The client's `Connection` semantics and the origin's are fully decoupled; you cannot influence the client-edge connection via origin headers.
- If your origin sends `Connection: close`, the CDN closes *that origin socket* but keeps serving clients from its edge — it does not propagate `close` to end users.
- Cloudflare and most CDNs will strip hop-by-hop headers (including anything named in `Connection`) at the edge; do not use the `Connection` list to pass data through a CDN.
- Ensure your origin's `keepAliveTimeout` exceeds the CDN's origin idle timeout to avoid the same race that plagues LBs.

## Cloud Deployment Considerations

- **AWS ALB/ELB:** Default idle timeout is 60s. Set Node `server.keepAliveTimeout = 65000` and `server.headersTimeout = 66000`. This is the documented fix for the well-known ALB 502 issue.
- **GCP HTTP(S) LB / Azure App Gateway:** Same principle — origin keep-alive timeout must exceed the LB's idle timeout.
- **API Gateways (AWS API Gateway, Kong, Apigee):** They terminate client connections and pool to the backend; connection management is theirs, not the client's.
- **Serverless (Lambda, Cloud Functions):** Connections are often not reusable across invocations in the traditional sense; keep-alive tuning applies to the *outbound* SDK/HTTP agents inside the function (enable `keepAlive` on the AWS SDK's HTTP agent to cut DynamoDB/S3 latency dramatically).

## Debugging

- **Chrome DevTools:** The Network panel's "Connection ID" column (enable via column settings) shows which requests share a socket — useful to confirm keep-alive reuse and to see HTTP/2 multiplexing (all share one ID). Note `Connection` itself won't appear as a request header for H2.
- **curl:** `curl -v --http1.1 https://api.example.com/orders` prints the `Connection` header on both sides. Use `curl -v --http1.1 -H 'Connection: close' ...` to force per-request close and observe teardown. `curl` reuses connections within a single invocation when you pass multiple URLs.
- **Postman / Bruno:** Both let you inspect response headers including `Connection`/`Keep-Alive`; they manage their own keep-alive agent, so behavior may differ from a browser.
- **Node.js:** Log `req.headers.connection` and `req.httpVersion` to see what actually arrived. Instrument `server.on('connection')` and socket `close` events to observe pool churn. `netstat -an | grep :3000` or `ss -tan` shows live vs TIME_WAIT sockets — a flood of new connections instead of reuse signals broken keep-alive (usually the Nginx `proxy_http_version`/`Connection ""` mistake).
- **Express logging:** Add a middleware logging `req.headers.connection`, `req.httpVersion`, and the socket's reuse count to correlate with 502 spikes.

## Best Practices

- Leave keep-alive **on** (the HTTP/1.1 default); never disable it as a blanket policy.
- Behind any load balancer/CDN, set `keepAliveTimeout` **above** the LB idle timeout and `headersTimeout` above that.
- On HTTP clients (Node, SDKs), always use a keep-alive `Agent` and **drain response bodies** to return sockets to the pool.
- In reverse proxies, use `proxy_http_version 1.1` + `keepalive` upstream + `Connection ""` to reuse upstream sockets; use `Connection "upgrade"` only on WebSocket locations.
- Never set `Connection` from browser code — it's forbidden and ignored.
- In custom proxies, strip `Connection` and every header it names before forwarding; never trust it to override protocol framing (smuggling defense).
- Don't branch application logic on `Connection` — it won't exist under HTTP/2/3.

## Related Headers

- **[Upgrade](./Upgrade.md)** — `Connection: Upgrade` is the mandatory companion token that tells the current hop the connection is switching protocols.
- **[Keep-Alive](../04-Response-Headers/Keep-Alive.md)** — tuning parameters (`timeout`, `max`) for a persistent connection; only meaningful alongside `Connection: keep-alive`.
- **[Transfer-Encoding](../10-Compression/Transfer-Encoding.md)** — hop-by-hop like `Connection`; their interaction is the smuggling surface.
- **[End-to-End vs Hop-by-Hop Headers](../01-Introduction/End-to-End-vs-Hop-by-Hop-Headers.md)** — the taxonomy `Connection` both belongs to and defines.
- **`Proxy-Connection`** — a nonstandard legacy variant some old proxies used; treat as deprecated.

## Decision Tree

```
Are you on HTTP/2 or HTTP/3?
├── Yes → Do NOT send Connection. It's forbidden. The protocol manages connections.
└── No (HTTP/1.1) →
    Are you writing browser fetch/XHR code?
    ├── Yes → Don't set Connection; it's ignored. The browser owns it.
    └── No →
        Are you shutting down / draining a node?
        ├── Yes → Send `Connection: close` to force clients off this socket.
        └── No →
            Are you a reverse proxy talking to an upstream?
            ├── WebSocket upstream → `Connection: upgrade` + forward Upgrade.
            ├── Normal upstream → `Connection: ""` + proxy_http_version 1.1 + keepalive.
            └── Otherwise → leave it alone; keep-alive default is correct.
```

## Mental Model

**`Connection` is a note to the person standing directly across from you — never to the final recipient.** Think of a relay race: `keep-alive` means "stay in position, I'm passing you the baton again"; `close` means "I'm leaving, don't wait for me." And the header's list feature is like tagging certain messages "for your eyes only, don't pass this on" — the runner reads them and drops them before handing off. Because it's addressed to *this* runner on *this* leg, forwarding it down the track is nonsense — which is exactly why every proxy peels it off and writes its own.
