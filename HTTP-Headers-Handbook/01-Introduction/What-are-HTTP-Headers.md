# What are HTTP Headers

## Quick Summary

HTTP headers are key–value metadata attached to every HTTP request and response. They are **not** the content you asked for (that's the body) — they are the instructions, context, and negotiation data that tell the other side *how to interpret, route, cache, secure, and process* the message. Almost every cross-cutting concern in web engineering — caching, compression, authentication, CORS, cookies, content negotiation, security policy — is expressed and controlled through headers. If the URL says *what* you want and the method says *what to do with it*, headers say *everything else about how the exchange should behave*.

## The mental model to hold

Think of an HTTP message as a shipped package:

- The **request line / status line** is the shipping label's primary instruction (`GET /users/42 HTTP/1.1`, or `200 OK`).
- The **headers** are the customs form, handling instructions, and manifest stapled to the outside: "fragile," "contents: JSON," "return to this address," "already paid," "keep refrigerated for 60 seconds."
- The **body** is what's inside the box.

Crucially, intermediaries (proxies, CDNs, load balancers, reverse proxies) can read and act on the *outside of the box* — the headers — often without ever opening it. That is the entire reason headers exist as a separate, structured channel: they let every hop in the network make decisions cheaply.

## Why metadata had to be separate from content

Early hypertext could have been a single blob: "here is the document." But the web needed answers to questions that have nothing to do with the document's bytes:

- *What format is this?* → `Content-Type: text/html; charset=utf-8`
- *How big is it?* → `Content-Length: 5120`
- *Can I cache it, and for how long?* → `Cache-Control: max-age=3600`
- *Who is asking, and are they allowed?* → `Authorization: Bearer …`
- *Which language / encoding does the client want?* → `Accept-Language: en-US`, `Accept-Encoding: gzip, br`
- *Is this the same version I already have?* → `ETag` / `If-None-Match`

You cannot answer these by inspecting the body, because to inspect the body you must first *download and parse* it — which defeats caching, defeats routing, and defeats security. Headers are the **out-of-band control plane** of HTTP. They are designed to be small, textual (in HTTP/1.1), and parseable before or without the body.

## A concrete example

Here is a real request/response pair. Notice how little of it is the actual "content."

**Request**

```http
GET /api/products/42 HTTP/1.1
Host: shop.example.com
Accept: application/json
Accept-Encoding: gzip, br
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6...
If-None-Match: "v3-abc123"
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) ...
```

**Response**

```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Encoding: br
Content-Length: 318
Cache-Control: public, max-age=60, stale-while-revalidate=300
ETag: "v4-def456"
Vary: Accept-Encoding, Authorization
Date: Mon, 07 Jul 2026 10:00:00 GMT

{"id":42,"name":"Wireless Mouse","price":29.99}
```

Only the last line is the "answer." Everything above it is headers coordinating format, compression, authentication, caching, and cache-key correctness. In production, **the headers are where the engineering lives.** A wrong `Cache-Control` can serve a logged-in user another user's private data from a CDN. A missing `Vary` can serve gzip bytes to a client that can't decompress them. A wrong `Content-Type` can turn a JSON API into an XSS vector. This handbook is fundamentally about getting these lines right.

## Headers are a shared vocabulary across many actors

The same header can be read and written by very different systems, each with its own agenda:

```mermaid
sequenceDiagram
    participant B as Browser
    participant CDN as CDN (edge)
    participant RP as Reverse Proxy (Nginx)
    participant APP as App (Express/Node)
    B->>CDN: GET /api/products/42<br/>Accept-Encoding: gzip, br<br/>Authorization: Bearer ...
    CDN->>RP: forwards + adds Via, X-Forwarded-For
    RP->>APP: forwards + adds X-Forwarded-Proto, X-Real-IP
    APP-->>RP: 200 + Cache-Control, ETag, Vary
    RP-->>CDN: passes response, may add/strip headers
    CDN-->>B: may serve from edge cache using Cache-Control + Vary
```

- The **browser** sets request headers automatically (`Host`, `User-Agent`, `Accept*`, `Cookie`, `Origin`) and *acts on* response headers (`Set-Cookie`, `Content-Type`, `Cache-Control`, CSP, CORS).
- The **CDN** reads `Cache-Control`/`Vary`/`ETag` to decide whether to serve from the edge, and adds its own diagnostic headers.
- The **reverse proxy** adds forwarding headers so the app knows the *real* client IP and scheme.
- The **application** produces the authoritative response headers that everything downstream obeys.

A header is only useful because all these actors agree on what it means. That agreement is codified in RFCs (chiefly RFC 9110 "HTTP Semantics," which superseded RFC 7230–7235, plus topic-specific RFCs for cookies, CORS, caching, etc.).

## What a header is *not*

- **Not the body / payload.** Headers describe the body; they are not it.
- **Not the URL or query string.** Those are part of the request target. (Though headers like `Host` combine with the path to form the effective request URL.)
- **Not guaranteed to survive intact.** Intermediaries legitimately add, remove, reorder, or rewrite headers. Never assume a header you sent arrives byte-for-byte, and never trust a header you received without knowing who could have set it (this is the root of `X-Forwarded-For` spoofing and host-header attacks).
- **Not case-sensitive in their names.** `Content-Type`, `content-type`, and `CONTENT-TYPE` are the same header. (Values may or may not be case-sensitive depending on the header.) In HTTP/2 and HTTP/3, header names are required to be lowercase on the wire — more on this in *HTTP Versions and Headers*.

## Where headers come from in your stack

As a full-stack JS developer you rarely write raw HTTP. Headers reach you through abstractions:

- **In the browser:** `fetch()` / `XMLHttpRequest` / `axios` let you set *some* request headers; the browser forces others (`Host`, `Origin`, `Cookie`, `Content-Length`, `User-Agent` — the "forbidden headers"). Response headers are read via `response.headers`, though CORS restricts which ones JS can see (see `Access-Control-Expose-Headers`).
- **In Node.js:** the `http`/`https` modules expose `req.headers` (a lowercased object) and `res.setHeader()` / `res.writeHead()`.
- **In Express:** `req.get('...')`, `req.headers`, `res.set()`, `res.type()`, `res.cookie()`, and middleware like `helmet`, `cors`, and `compression` that manage whole header families for you.

Understanding what these abstractions set *for you* — and what they let you override — is half of using headers correctly in production. Every header page in this handbook shows the raw HTTP and the Express/Node/React code side by side so you can connect the two.

## Why this matters more than it looks

Most production incidents that "make no sense" are header incidents:

- A deploy "works locally" but returns stale assets in prod → caching headers + CDN.
- An API works in Postman but fails in the browser → CORS headers.
- Login "randomly logs users out" → `Set-Cookie` attributes (`SameSite`, `Secure`, `Domain`).
- A file download opens in the tab instead of saving → `Content-Disposition`.
- Users behind a proxy all appear to have one IP → `X-Forwarded-For` + `trust proxy`.
- A page mysteriously won't load in an iframe, or fonts fail cross-origin → CSP / `X-Frame-Options` / CORP.

None of these are bugs in your business logic. They are header configuration. That is why a staff engineer treats headers as first-class infrastructure, not an afterthought.

## Related Reading

- [Anatomy of an HTTP Message](./Anatomy-of-an-HTTP-Message.md) — exactly where headers sit on the wire.
- [Request vs Response Headers](./Request-vs-Response-Headers.md) — who sets what.
- [End-to-End vs Hop-by-Hop Headers](./End-to-End-vs-Hop-by-Hop-Headers.md) — which headers intermediaries may consume.
- [Header Categories](../02-Core-Concepts/Header-Categories.md) — the functional taxonomy used throughout this book.

## Mental Model

**Headers are the shipping paperwork of the web.** The body is the cargo; the headers are every stamp, label, and instruction that lets browsers, proxies, CDNs, and servers handle the cargo correctly *without unpacking it*. Get the paperwork wrong and the right cargo goes to the wrong place, gets cached for the wrong person, or gets rejected at the border — even though the cargo itself was perfect.
