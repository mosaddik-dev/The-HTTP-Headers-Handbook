# End-to-End vs Hop-by-Hop Headers

## Quick Summary

This is a classification orthogonal to request/response: it's about **how far a header is allowed to travel**. **End-to-end headers** are meant for the final recipient and must be forwarded unchanged by every intermediary (proxy, CDN, reverse proxy) along the way. **Hop-by-hop headers** are meaningful only for a *single* connection between two adjacent nodes and **must be consumed and not forwarded** — they describe *this TCP hop*, not the overall request. Confusing the two leads to broken keep-alive, leaked upgrade tokens, and subtle proxy bugs. The set of hop-by-hop headers is small and mostly about connection management.

## Why the distinction exists

An HTTP request rarely travels over a single connection. A realistic path:

```
Browser ──conn1──▶ CDN edge ──conn2──▶ Reverse proxy (Nginx) ──conn3──▶ App (Node)
```

Each `──connN──▶` is a separate TCP (or QUIC) connection. Some headers describe the *overall* request/response (e.g., "the body is JSON," "cache this for 60s," "here's my auth token") — those must survive all the way to the app and back. Other headers describe only *how to manage this specific socket* (e.g., "keep this connection alive for 5 more requests," "I want to switch this connection to WebSocket") — those are nonsense to forward, because the next hop is a *different* connection with its own management needs.

So HTTP defines that intermediaries must **strip hop-by-hop headers before forwarding** and may add their own for the next hop.

## The hop-by-hop headers

RFC 7230 (now RFC 9110/9112) defines these as hop-by-hop by default:

- `Connection`
- `Keep-Alive`
- `Proxy-Authenticate`
- `Proxy-Authorization`
- `TE`
- `Trailer`
- `Transfer-Encoding`
- `Upgrade`

Additionally — and this is the clever part — **the `Connection` header names further hop-by-hop headers for that specific message.** Whatever tokens you list in `Connection` must also be treated as hop-by-hop and stripped:

```http
Connection: keep-alive, X-Internal-Trace
X-Internal-Trace: abc123
```

Here `X-Internal-Trace` is declared hop-by-hop *for this message*; a conforming proxy removes both `Connection` and `X-Internal-Trace` before forwarding. This mechanism lets you attach a header that should never leak past the immediate next hop.

**Everything else is end-to-end** — `Content-Type`, `Cache-Control`, `Authorization`, `Set-Cookie`, `ETag`, all the security and CORS headers, etc. Intermediaries forward them (though CDNs/proxies may still add or, with explicit config, modify some).

## Hop-by-hop headers, one by one

- **`Connection`** — controls the current connection: `keep-alive` (reuse the socket), `close` (close after this message), or a list of header names to treat as hop-by-hop. Meaningless to the final app; managed per hop.
- **`Keep-Alive`** — tuning for persistent connections (`timeout`, `max`). Per hop.
- **`Transfer-Encoding`** — how the body is framed *on this connection* (`chunked`). The next hop may re-frame it entirely (e.g., a proxy buffers a chunked response and forwards it with a `Content-Length`). This is why `Transfer-Encoding` is hop-by-hop and why mixing it with `Content-Length` enables smuggling.
- **`TE`** — the transfer-codings the client will accept on this hop (e.g., `trailers`).
- **`Trailer`** — lists header fields that appear *after* a chunked body. Tied to the chunked framing of this hop.
- **`Upgrade`** — request to switch protocols *on this connection* (HTTP→WebSocket, HTTP/1.1→h2c). Only meaningful between the two endpoints of the socket being upgraded. See [Upgrade](../03-Request-Headers/Upgrade.md).
- **`Proxy-Authenticate` / `Proxy-Authorization`** — authentication *to the proxy itself*, not to the origin. By definition per hop.

## Why this matters in production

### 1. WebSocket / SSE through a reverse proxy

The single most common place this bites developers: proxying WebSockets. Because `Upgrade` and `Connection` are hop-by-hop, Nginx **will not forward them by default** — so the upgrade never reaches your Node app and the WebSocket handshake fails. You must explicitly re-add them for the upstream hop:

```nginx
location /socket.io/ {
    proxy_pass http://app_upstream;
    proxy_http_version 1.1;               # keep-alive + upgrade require HTTP/1.1 upstream
    proxy_set_header Upgrade $http_upgrade;      # re-inject the (otherwise stripped) hop-by-hop header
    proxy_set_header Connection "upgrade";       # tell upstream this hop wants to upgrade
    proxy_set_header Host $host;                  # end-to-end, but good hygiene to set explicitly
    proxy_read_timeout 3600s;                     # WebSockets are long-lived; don't time them out
}
```

Every line of the upgrade dance exists *because* `Upgrade`/`Connection` are hop-by-hop and would otherwise be dropped between the proxy and the app.

### 2. Chunked → Content-Length re-framing

Your Node app streams a `Transfer-Encoding: chunked` response. A buffering proxy or CDN may collect the whole body and forward it to the browser with `Content-Length` instead. That's *legal* precisely because `Transfer-Encoding` is hop-by-hop — the framing is renegotiated per hop. Don't assume the client sees the same framing you emitted.

### 3. Don't smuggle secrets in hop-by-hop-declared headers by accident

If you add `Connection: close, X-Debug` you've just told every proxy to strip `X-Debug`. Conversely, if you *want* a header to reach the origin, never list it in `Connection`.

### 4. HTTP/2 and HTTP/3 outright ban most of them

In HTTP/2/3 the connection is managed by the protocol's own framing, so the connection-specific headers are **forbidden**: `Connection`, `Keep-Alive`, `Transfer-Encoding`, `Upgrade`, and `Proxy-Connection` must not appear (`TE` may only carry `trailers`). Sending them is a protocol error (`PROTOCOL_ERROR`). Protocol switching that used `Upgrade` in HTTP/1.1 (WebSockets, h2c) uses different mechanisms in H2/H3 (the extended CONNECT method, or ALPN at the TLS layer). See [HTTP Versions and Headers](./HTTP-Versions-and-Headers.md).

## How to reason about a header you're unsure about

Ask: *"Does this describe the request/response as a whole, or just this one socket?"*

- "The body is gzip-compressed" → about the whole representation → **end-to-end** (`Content-Encoding`, note: NOT `Transfer-Encoding`).
- "Reuse this socket for 5 more requests" → about the socket → **hop-by-hop** (`Keep-Alive`).
- "Cache this for 60 seconds" → about the resource → **end-to-end** (`Cache-Control`).
- "Switch this connection to WebSocket" → about the socket → **hop-by-hop** (`Upgrade`).

Note the deliberate pairing: **`Content-Encoding` is end-to-end** (compression is a property of the resource and survives all hops) while **`Transfer-Encoding` is hop-by-hop** (framing is a property of the connection). Mixing them up is a classic error — see [Content-Encoding](../10-Compression/Content-Encoding.md) vs [Transfer-Encoding](../10-Compression/Transfer-Encoding.md).

## Debugging

- **Symptom: WebSocket/SSE works direct to Node but 400/502 through Nginx** → the proxy stripped `Upgrade`/`Connection`; add them back (above).
- **Symptom: a custom header vanishes at the app** → check whether something listed it in `Connection`, or an intermediary treats it as hop-by-hop.
- **Symptom: `Transfer-Encoding` present in HTTP/2** → misconfigured translation layer; H2 forbids it.
- **Inspect with curl:** `curl -v --http1.1 ...` shows `Connection`, `Upgrade`, etc., on each side; compare what you sent vs what the app logged (`req.rawHeaders`).

## Related Reading

- [Connection](../03-Request-Headers/Connection.md), [Upgrade](../03-Request-Headers/Upgrade.md), [Transfer-Encoding](../10-Compression/Transfer-Encoding.md)
- [Proxies Overview](../14-Proxies/Proxies-Overview.md) and [Reverse Proxy Overview](../16-Reverse-Proxies/Reverse-Proxy-Overview.md)
- [HTTP Versions and Headers](./HTTP-Versions-and-Headers.md) — why H2/H3 ban the connection headers.

## Mental Model

**End-to-end headers are a sealed letter addressed to the final recipient; hop-by-hop headers are sticky notes between adjacent mail carriers.** The sealed letter (`Authorization`, `Content-Type`, `Cache-Control`) must be handed along untouched until it reaches the addressee. The sticky notes ("hand-off complete, keep this route open," "switch to the express lane") are meaningful only between two carriers and are peeled off and rewritten at every handoff. A carrier who forwards a sticky note by mistake confuses the next carrier — which is exactly what a misbehaving proxy does when it leaks `Connection` or `Upgrade`.
