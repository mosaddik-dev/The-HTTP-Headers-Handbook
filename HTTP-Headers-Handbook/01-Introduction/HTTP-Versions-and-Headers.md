# HTTP Versions and Headers

Every chapter in this handbook talks about "headers" as if they were a single, stable concept. Semantically they are — `Content-Type` means the same thing everywhere. But *how* those headers are represented, framed, compressed, and constrained changed dramatically across HTTP/1.0, HTTP/1.1, HTTP/2, and HTTP/3. As a full-stack engineer you rarely serialize headers by hand — the runtime does it — but the version determines which headers are legal, which are banned, how connection reuse works, and why some "obvious" headers (`Connection`, `Transfer-Encoding`) silently vanish or cause errors on newer protocols.

This chapter is the bridge between the *abstract* header model (see [What are HTTP Headers](./What-are-HTTP-Headers.md) and [Anatomy of an HTTP Message](./Anatomy-of-an-HTTP-Message.md)) and the *wire reality* across versions. The recurring theme: **the header semantics stay constant; the transport and the rules around connection-management headers change.**

## HTTP/1.0 vs HTTP/1.1

HTTP/1.0 and HTTP/1.1 share the same textual, human-readable format: a request line, CRLF-separated `Name: Value` headers, a blank line, then the body. The differences that matter for headers:

### `Host` became mandatory

HTTP/1.0 had no `Host` header — one IP served one site. HTTP/1.1 **requires** `Host` on every request, which is what makes name-based virtual hosting possible: one server, one IP, many domains, disambiguated by `Host`.

```http
GET /index.html HTTP/1.1
Host: www.example.com
```

Omit `Host` on an HTTP/1.1 request and a compliant server responds `400 Bad Request`. This is why `Host` is one of the browser-owned, forbidden-to-script headers from [How Browsers Process Headers](./How-Browsers-Process-Headers.md) — it is structurally required.

### Persistent connections (keep-alive)

HTTP/1.0 closed the TCP connection after every request/response by default; reuse required an explicit `Connection: keep-alive`. HTTP/1.1 **inverted the default** — connections persist unless a party sends `Connection: close`. Persistence avoids paying the TCP (and TLS) handshake cost per request, but HTTP/1.1 still processes one request at a time per connection (head-of-line blocking), which is precisely the limitation HTTP/2 was built to remove.

`Connection` and `Keep-Alive` are **hop-by-hop** headers — they describe *this* TCP connection, not the end-to-end message, which is why a proxy must consume and regenerate them (see [End-to-End vs Hop-by-Hop Headers](./End-to-End-vs-Hop-by-Hop-Headers.md)).

### Chunked transfer encoding

HTTP/1.1 introduced `Transfer-Encoding: chunked`, letting a server stream a body of unknown length by prefixing each chunk with its size and terminating with a zero-length chunk (see the framing discussion in [How Servers Process Headers](./How-Servers-Process-Headers.md)). HTTP/1.0 had no chunked encoding — a body of unknown length could only be delimited by closing the connection.

## HTTP/2: same headers, completely different wire

HTTP/2 keeps HTTP's *semantics* — same methods, same status codes, same header *names and meanings* — but replaces the textual wire format with a **binary framing layer** and multiplexes many concurrent streams over one TCP connection. For header handling specifically, five things change and matter to developers.

### 1. Binary framing and multiplexing

There is no more textual header block on the wire. Requests and responses are split into typed binary frames (`HEADERS`, `DATA`, ...) carried on independent streams over a single connection. This eliminates HTTP/1.1's head-of-line blocking *at the HTTP layer* — a slow response no longer blocks others on the same connection. (TCP-level head-of-line blocking remains, which is what HTTP/3 fixes.) You never see frames from application code; your `req.headers` looks identical.

### 2. HPACK header compression

HTTP/1.1 sent full, uncompressed header text on every single request — and browsers send a *lot* of repetitive headers (`User-Agent`, `Cookie`, `Accept`) unchanged across hundreds of requests. HTTP/2 compresses headers with **HPACK**: a static table of common header names/values, a dynamic table that both peers build up as the connection lives, and Huffman coding for literals. After the first request on a connection, repeated headers cost a byte or two instead of hundreds.

HPACK has a security consequence — the shared compression state across requests is what made the **CRIME/HPACK-adjacent** attacks relevant, and it is why HPACK deliberately does *not* compress across the security boundary the way naive gzip-of-headers would. As a developer you get the win for free; just know that "headers are basically free after the first request" only holds within a single connection.

### 3. Lowercase header names are mandatory

In HTTP/1.1, header names are case-insensitive but any casing is legal on the wire. In HTTP/2, header field names **must be lowercase** — sending `Content-Type` instead of `content-type` is a protocol error. This is why Node lowercases `req.headers` (see [How Servers Process Headers](./How-Servers-Process-Headers.md)): it matches the h2 wire requirement and papers over the h1 variability. If you ever hand-build h2 headers, uppercase names will be rejected.

### 4. Pseudo-headers replace the request/status line

HTTP/1.1's request line (`GET /path HTTP/1.1`) and status line (`HTTP/1.1 200 OK`) do not exist in HTTP/2. That information moves into **pseudo-headers** — special fields whose names begin with `:` and which must appear *before* all regular headers in the `HEADERS` frame.

Request pseudo-headers:

- **`:method`** — `GET`, `POST`, ... (was the request-line verb)
- **`:path`** — `/orders?id=1` (path + query)
- **`:scheme`** — `https`
- **`:authority`** — the host + optional port; **replaces the `Host` header**. (`Host` may still appear for compatibility, but `:authority` is authoritative.)

Response pseudo-header:

- **`:status`** — `200`, `404`, ... the *only* response pseudo-header.

```http
:method: GET
:scheme: https
:authority: api.example.com
:path: /orders?id=1
accept: application/json
```

Pseudo-headers are entirely browser/runtime-managed and never visible to your JS — they are the h2 encoding of things the browser already owned in h1 (`Host`, the method, the URL).

### 5. No reason phrase

HTTP/1.1 status lines carried a human-readable *reason phrase* — `200 OK`, `404 Not Found`. HTTP/2 dropped it entirely; there is only `:status: 404`. The phrase was never machine-significant, so nothing of value was lost, but code that parses or depends on the reason phrase (rare, but it exists in old logging) has nothing to read on h2.

### Banned connection-management headers in HTTP/2

This is the change that bites developers most. Because HTTP/2 manages the connection itself through its framing layer, the HTTP/1.1 **hop-by-hop / connection-management headers are forbidden**:

- **`Connection`**
- **`Keep-Alive`**
- **`Transfer-Encoding`** (and specifically the `chunked` value — framing is done by `DATA` frames now)
- **`Upgrade`**
- **`Proxy-Connection`**
- and `Connection` may not even *name* other headers to remove.

Sending any of these on HTTP/2 is a **protocol error** and the stream (or connection) is reset. The single legal exception is `TE`, and only with the value `trailers`.

Why does this matter to you even though your runtime handles framing? Two real scenarios:

1. **Copying headers between protocols.** A proxy or a piece of middleware that blindly forwards an HTTP/1.1 request's headers (including `Connection`, `Transfer-Encoding`) onto an HTTP/2 upstream will produce a protocol error. Correct proxies *strip* hop-by-hop headers at the boundary — this is the [End-to-End vs Hop-by-Hop](./End-to-End-vs-Hop-by-Hop-Headers.md) rule made mandatory.
2. **Hand-setting `Connection: keep-alive` "for performance."** It is meaningless on h2 (the connection is always persistent and multiplexed) and, if it reaches the h2 layer, harmful.

### Server push is deprecated

HTTP/2 originally shipped **server push** — the server could preemptively send resources the client had not requested. In practice it was hard to use correctly (you push things the client already cached, wasting bandwidth), and browsers have **removed support for it**. The modern replacement is **`103 Early Hints`** with `Link: rel=preload` headers, which lets the server *hint* what to fetch and lets the client decide — cache-aware and far more effective. Do not build on server push; use Early Hints (see the preload-scanner discussion in [How Browsers Process Headers](./How-Browsers-Process-Headers.md)).

## HTTP/3: HTTP/2 semantics over QUIC

HTTP/3 keeps the *entire* HTTP/2 header model — pseudo-headers, lowercase names, banned connection headers, `:status`, no reason phrase — and changes the *transport underneath*. Instead of running over TCP + TLS, it runs over **QUIC**, a transport built on UDP with TLS 1.3 baked in.

The developer-visible motivation: TCP has *transport-level* head-of-line blocking. Even though HTTP/2 multiplexes streams, a single lost TCP segment stalls *all* streams because TCP delivers bytes in order. QUIC gives each stream independent delivery, so one lost packet only stalls its own stream. Combined with a faster (often 0-RTT) handshake, HTTP/3 is meaningfully faster on lossy networks (mobile).

For headers, the one substantive change is compression: HPACK depended on TCP's guaranteed in-order delivery, which QUIC does not provide. HTTP/3 therefore uses **QPACK** — the same idea (static + dynamic table + Huffman) redesigned to tolerate out-of-order stream delivery without letting header decoding on one stream block another. From your code's perspective QPACK is invisible; the point is only that "why not just reuse HPACK?" has a concrete answer: QUIC's independent streams broke HPACK's ordering assumption.

Everything else you learned for HTTP/2 headers applies unchanged to HTTP/3.

## What changes for developers vs. what stays the same

**Stays the same** across all versions:

- Header *names and semantics* (`Content-Type`, `Cache-Control`, `Set-Cookie`, `Authorization`, CORS headers) — identical everywhere.
- `req.headers` in Node, `res.set()` in Express — the runtime abstracts the wire format entirely. Your application code is version-agnostic.
- Methods, status codes, the request/response model.

**Changes** (and where it leaks into your world):

- Connection-management headers (`Connection`, `Keep-Alive`, `Transfer-Encoding`, `Upgrade`) are legal in h1, **banned** in h2/h3. Leaks into proxy/middleware code and hand-set headers.
- Header names must be lowercase on h2/h3. Leaks if you build headers manually or compare case-sensitively.
- Framing: `Content-Length`/`chunked` in h1 vs. binary `DATA` frames in h2/h3. Leaks into any manual body-framing.
- Performance model: connection reuse (h1.1) → multiplexing (h2) → per-stream independence (h3). Leaks into how you think about batching, connection pooling, and domain sharding (which is now an *anti-pattern* on h2/h3).

## Comparison table

| Aspect | HTTP/1.0 | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|---|
| Wire format | Text | Text | Binary frames | Binary frames |
| Transport | TCP | TCP | TCP + TLS | QUIC (UDP) + TLS 1.3 |
| `Host` required | No | **Yes** | Via `:authority` | Via `:authority` |
| Persistent conn (default) | No (opt-in keep-alive) | **Yes** (opt-out `close`) | Always (multiplexed) | Always (multiplexed) |
| Multiplexing | No | No | **Yes** (one TCP conn) | **Yes** (independent streams) |
| Head-of-line blocking | Per request | HTTP-layer | Removed at HTTP layer; TCP-layer remains | **Removed** |
| Header compression | None | None | **HPACK** | **QPACK** |
| Header name case | Insensitive | Insensitive | **Must be lowercase** | **Must be lowercase** |
| Request/status line | Text lines | Text lines | **Pseudo-headers** (`:method`,`:path`,`:scheme`,`:authority`,`:status`) | Pseudo-headers |
| Reason phrase | Yes | Yes | **No** | No |
| `Connection`/`Keep-Alive`/`Transfer-Encoding`/`Upgrade` | Allowed | Allowed (hop-by-hop) | **Banned** | **Banned** |
| Chunked transfer encoding | No | Yes | N/A (frames) | N/A (frames) |
| Server push | No | No | Yes (deprecated/removed) | Discouraged |
| Protocol negotiation | — | `Upgrade` / prior knowledge | ALPN (over TLS) | ALPN + `Alt-Svc` |

## Protocol switching: WebSockets and CONNECT without `Upgrade`

HTTP/1.1 switches protocols with the `Upgrade` header and a `101 Switching Protocols` response — this is exactly how a **WebSocket** handshake works:

```http
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

But `Upgrade` and `Connection` are **banned in HTTP/2 and HTTP/3**. So how do WebSockets work over h2/h3? Through **Extended CONNECT** (RFC 8441): the server advertises support with a `SETTINGS_ENABLE_CONNECT_PROTOCOL` setting, and the client opens a stream with `:method: CONNECT` plus a **`:protocol: websocket`** pseudo-header. The WebSocket then runs inside that single h2/h3 stream — multiplexed alongside normal requests instead of monopolizing a whole TCP connection.

```http
:method: CONNECT
:protocol: websocket
:scheme: https
:authority: example.com
:path: /chat
sec-websocket-version: 13
```

Two related negotiation mechanisms round this out:

- **ALPN (Application-Layer Protocol Negotiation)** is a TLS extension: during the TLS handshake, client and server agree on the protocol (`h2`, `http/1.1`, `h3`). This is how a browser and server pick HTTP/2 *without* an in-band `Upgrade` dance — the negotiation happens in TLS, before the first HTTP byte. HTTP/2 over cleartext (`h2c`) used `Upgrade`, but browsers only do h2 over TLS, so ALPN is the real-world path.
- **`Alt-Svc`** lets a server running HTTP/1.1 or /2 advertise "I also speak HTTP/3 at this endpoint," so the client can switch to h3 on the next connection. Since h3 is over UDP/QUIC and cannot be reached by upgrading a TCP connection, `Alt-Svc` (or the `DNS HTTPS` record) is how discovery happens.

The through-line: the *idea* of "switch this connection to another protocol" survives, but the `Upgrade` **header** does not survive into h2/h3. It is replaced by pseudo-header-based Extended CONNECT and out-of-band negotiation (ALPN, `Alt-Svc`). This is the header-namespace consequence of the connection-management ban.

## Mental Model

Picture the header model as **stable cargo carried by increasingly sophisticated trucks.**

- The **cargo** — header names and their meanings, methods, status codes — is identical in every version. Your Node/Express code reads and writes that cargo the same way regardless of the truck. This is why you almost never write version-specific header code.
- The **truck** changes: HTTP/1.0 is a truck that makes a round trip per package; HTTP/1.1 keeps the engine running between packages (keep-alive) but still one-package-at-a-time; HTTP/2 is a binary, multi-lane truck (framing + multiplexing + HPACK) that requires lowercase labels and puts the shipping manifest on special `:`-prefixed tags (pseudo-headers); HTTP/3 is the same multi-lane truck riding on a new road (QUIC/UDP) with QPACK labels, so one pothole doesn't stop every lane.
- The **rule that leaks into your code**: connection-management headers (`Connection`, `Keep-Alive`, `Transfer-Encoding`, `Upgrade`) describe the *truck*, not the *cargo* — so newer trucks that manage themselves **ban** them. Protocol switching (WebSockets) is no longer an `Upgrade` sticker on the cargo; it's a manifest instruction (Extended CONNECT + `:protocol`) negotiated up front (ALPN, `Alt-Svc`).

When something breaks across versions, ask: is this an *end-to-end* header (cargo — safe everywhere) or a *hop-by-hop/connection* header (truck — version-constrained)? That single question, grounded in [End-to-End vs Hop-by-Hop Headers](./End-to-End-vs-Hop-by-Hop-Headers.md), explains almost every version-related header error you will hit.
