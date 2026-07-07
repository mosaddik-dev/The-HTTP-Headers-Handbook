# HTTP Headers Cheat Sheet

> One-screen-per-category lookup for every header in this handbook. **Dir** = direction: `req` (client → server), `res` (server → client), `both`. Values shown are *typical* — see each header's page for the full grammar, edge cases, and production guidance.
>
> Cross-category headers (e.g. `ETag` is both caching and conditional; `Vary` is caching and negotiation; `Accept-Ranges` is response-general and range) are listed under their primary category and cross-referenced. Use Ctrl/⌘-F to jump.

**Jump to:** [Security](#security) · [Caching](#caching) · [Authentication](#authentication) · [CORS](#cors) · [Compression](#compression--transfer) · [Content Negotiation](#content-negotiation) · [Conditional](#conditional-requests) · [Range](#range-requests) · [Proxy / Forwarding](#proxy--forwarding) · [Cookies](#cookies) · [Request-general](#request-general) · [Response-general](#response-general)

---

## Security

| Header | Dir | Purpose | Typical value | Page |
|---|---|---|---|---|
| Strict-Transport-Security | res | Force HTTPS for future requests (HSTS) | `max-age=63072000; includeSubDomains; preload` | [→](../05-Security-Headers/Strict-Transport-Security.md) |
| Content-Security-Policy | res | Whitelist sources for scripts/styles/etc; blunt XSS | `default-src 'self'; script-src 'self' 'nonce-r4nd0m'` | [→](../05-Security-Headers/Content-Security-Policy.md) |
| X-Content-Type-Options | res | Stop MIME-sniffing; trust declared `Content-Type` | `nosniff` | [→](../05-Security-Headers/X-Content-Type-Options.md) |
| X-Frame-Options | res | Legacy clickjacking guard (superseded by CSP `frame-ancestors`) | `DENY` / `SAMEORIGIN` | [→](../05-Security-Headers/X-Frame-Options.md) |
| Referrer-Policy | res | Control how much `Referer` leaks cross-origin | `strict-origin-when-cross-origin` | [→](../05-Security-Headers/Referrer-Policy.md) |
| Permissions-Policy | res | Allow/deny powerful browser features (camera, geo…) | `geolocation=(), camera=()` | [→](../05-Security-Headers/Permissions-Policy.md) |
| Cross-Origin-Opener-Policy | res | Isolate browsing context; enable cross-origin isolation | `same-origin` | [→](../05-Security-Headers/Cross-Origin-Opener-Policy.md) |
| Cross-Origin-Embedder-Policy | res | Require CORP/CORS on all subresources (Spectre defense) | `require-corp` | [→](../05-Security-Headers/Cross-Origin-Embedder-Policy.md) |
| Cross-Origin-Resource-Policy | res | Restrict who may embed this resource | `same-origin` / `cross-origin` | [→](../05-Security-Headers/Cross-Origin-Resource-Policy.md) |
| X-DNS-Prefetch-Control | res | Toggle browser DNS prefetching | `off` / `on` | [→](../05-Security-Headers/X-DNS-Prefetch-Control.md) |

**See also:** [Referer](../03-Request-Headers/Referer.md), [Origin](../03-Request-Headers/Origin.md), [Sec-Fetch-*](../03-Request-Headers/Sec-Fetch.md), [Set-Cookie](../08-Cookies/Set-Cookie.md) (`Secure`/`HttpOnly`/`SameSite`).

---

## Caching

| Header | Dir | Purpose | Typical value | Page |
|---|---|---|---|---|
| Cache-Control | both | Master switch: storability, freshness, revalidation | `public, max-age=31536000, immutable` | [→](../06-Caching-Headers/Cache-Control.md) |
| Expires | res | Legacy absolute expiry date (superseded by `max-age`) | `Wed, 21 Oct 2026 07:28:00 GMT` | [→](../06-Caching-Headers/Expires.md) |
| ETag | res | Strong/weak content validator for revalidation | `"9f2c-a1b2c3"` / `W/"v7"` | [→](../06-Caching-Headers/ETag.md) |
| Last-Modified | res | Date-based (weaker) validator | `Tue, 15 Nov 2025 12:45:26 GMT` | [→](../06-Caching-Headers/Last-Modified.md) |
| Age | res | Seconds a shared cache has held the response | `120` | [→](../06-Caching-Headers/Age.md) |
| Vary | res | Which request headers form the cache key | `Accept-Encoding, Authorization` | [→](../06-Caching-Headers/Vary.md) |

**See also:** [If-None-Match](../12-Conditional-Requests/If-None-Match.md) / [If-Modified-Since](../12-Conditional-Requests/If-Modified-Since.md) (revalidation), [CDN Caching Overview](../15-CDNs/CDN-Caching-Overview.md), [Date](../04-Response-Headers/Date.md) (age math).

---

## Authentication

| Header | Dir | Purpose | Typical value | Page |
|---|---|---|---|---|
| Authorization | req | Carry credentials to the origin | `Bearer eyJhbGc…` / `Basic dXNlcjpwYXNz` | [→](../09-Authentication/Authorization.md) |
| WWW-Authenticate | res | Challenge the client (which scheme + realm) — sent with `401` | `Bearer realm="api", error="invalid_token"` | [→](../09-Authentication/WWW-Authenticate.md) |
| Proxy-Authenticate | res | Proxy's auth challenge — sent with `407` | `Basic realm="corp-proxy"` | [→](../14-Proxies/Proxies-Overview.md) |
| Proxy-Authorization | req | Credentials for a proxy (hop-by-hop) | `Basic dXNlcjpwYXNz` | [→](../14-Proxies/Proxies-Overview.md) |

**See also:** [Authentication Overview](../09-Authentication/Authentication-Overview.md), [Cookie](../08-Cookies/Cookie.md) / [Set-Cookie](../08-Cookies/Set-Cookie.md) (session auth), [Vary: Authorization](../06-Caching-Headers/Vary.md) (avoid cross-user cache leaks).

---

## CORS

| Header | Dir | Purpose | Typical value | Page |
|---|---|---|---|---|
| Origin | req | The origin initiating a cross-origin request | `https://app.example.com` | [→](../03-Request-Headers/Origin.md) |
| Access-Control-Request-Method | req | (Preflight) method the real request will use | `PUT` | [→](../07-CORS/Access-Control-Request-Method.md) |
| Access-Control-Request-Headers | req | (Preflight) custom headers the real request will send | `content-type, x-api-key` | [→](../07-CORS/Access-Control-Request-Headers.md) |
| Access-Control-Allow-Origin | res | Which origin may read the response | `https://app.example.com` / `*` | [→](../07-CORS/Access-Control-Allow-Origin.md) |
| Access-Control-Allow-Methods | res | (Preflight) methods allowed on the resource | `GET, POST, PUT, DELETE` | [→](../07-CORS/Access-Control-Allow-Methods.md) |
| Access-Control-Allow-Headers | res | (Preflight) request headers the client may send | `Content-Type, Authorization` | [→](../07-CORS/Access-Control-Allow-Headers.md) |
| Access-Control-Allow-Credentials | res | Allow cookies/credentials on cross-origin reads | `true` | [→](../07-CORS/Access-Control-Allow-Credentials.md) |
| Access-Control-Expose-Headers | res | Response headers JS may read via `getResponseHeader` | `ETag, X-Total-Count` | [→](../07-CORS/Access-Control-Expose-Headers.md) |
| Access-Control-Max-Age | res | How long (s) the browser may cache the preflight | `600` | [→](../07-CORS/Access-Control-Max-Age.md) |

**See also:** [CORS Overview](../07-CORS/CORS-Overview.md), [Vary: Origin](../06-Caching-Headers/Vary.md) (reflected-origin caching), [Cross-Origin-Resource-Policy](../05-Security-Headers/Cross-Origin-Resource-Policy.md).

---

## Compression & Transfer

| Header | Dir | Purpose | Typical value | Page |
|---|---|---|---|---|
| Accept-Encoding | req | Compression algorithms the client accepts | `gzip, br, zstd` | [→](../10-Compression/Accept-Encoding.md) |
| Content-Encoding | res | Compression applied to the body | `br` / `gzip` | [→](../10-Compression/Content-Encoding.md) |
| Transfer-Encoding | res | Hop-by-hop framing of the body (chunked) | `chunked` | [→](../10-Compression/Transfer-Encoding.md) |

**See also:** [Vary: Accept-Encoding](../06-Caching-Headers/Vary.md) (keep compressed variants distinct), [Content-Length](../04-Response-Headers/Content-Length.md) (mutually exclusive with `chunked`), `Cache-Control: no-transform` (block proxy recompression).

---

## Content Negotiation

| Header | Dir | Purpose | Typical value | Page |
|---|---|---|---|---|
| Accept | req | Media types the client prefers (+ q-values) | `text/html, application/json;q=0.9` | [→](../03-Request-Headers/Accept.md) |
| Accept-Language | req | Preferred natural languages (+ q-values) | `en-US, en;q=0.8, fr;q=0.5` | [→](../03-Request-Headers/Accept-Language.md) |
| Accept-Encoding | req | Preferred compression (see Compression) | `gzip, br` | [→](../10-Compression/Accept-Encoding.md) |
| Content-Type | both | Media type (+ charset) of the body | `application/json; charset=utf-8` | [→](../04-Response-Headers/Content-Type.md) |
| Vary | res | Which negotiation headers changed the response | `Accept, Accept-Language, Accept-Encoding` | [→](../06-Caching-Headers/Vary.md) |

**See also:** [Content Negotiation Overview](../11-Content-Negotiation/Content-Negotiation-Overview.md), [quality values / q-factors](./Glossary.md#quality-value-q-value).

---

## Conditional Requests

| Header | Dir | Purpose | Typical value | Page |
|---|---|---|---|---|
| If-None-Match | req | Send validator; get `304` if unchanged (GET) or `412` (writes) | `"9f2c-a1b2c3"` / `*` | [→](../12-Conditional-Requests/If-None-Match.md) |
| If-Modified-Since | req | Get `304` if not modified since date | `Tue, 15 Nov 2025 12:45:26 GMT` | [→](../12-Conditional-Requests/If-Modified-Since.md) |
| If-Match | req | Optimistic-lock writes; `412` if validator differs | `"v7"` | [→](../12-Conditional-Requests/Conditional-Requests-Overview.md) |
| If-Unmodified-Since | req | Proceed only if unchanged since date; else `412` | `Tue, 15 Nov 2025 12:45:26 GMT` | [→](../12-Conditional-Requests/Conditional-Requests-Overview.md) |
| ETag | res | Validator the conditionals compare against | `"9f2c-a1b2c3"` | [→](../06-Caching-Headers/ETag.md) |
| Last-Modified | res | Date validator for `If-Modified-Since`/`-Unmodified-Since` | `Tue, 15 Nov 2025 12:45:26 GMT` | [→](../06-Caching-Headers/Last-Modified.md) |

**See also:** [Conditional Requests Overview](../12-Conditional-Requests/Conditional-Requests-Overview.md), [If-Range](../13-Range-Requests/Range-Requests-Overview.md) (conditional + range).

---

## Range Requests

| Header | Dir | Purpose | Typical value | Page |
|---|---|---|---|---|
| Range | req | Request byte range(s) of a resource | `bytes=0-1023` / `bytes=1024-` | [→](../13-Range-Requests/Range.md) |
| Accept-Ranges | res | Advertise range support (or refuse) | `bytes` / `none` | [→](../04-Response-Headers/Accept-Ranges.md) |
| Content-Range | res | Which bytes are in a `206` (or `*/len` on `416`) | `bytes 0-1023/146515` | [→](../13-Range-Requests/Content-Range.md) |
| If-Range | req | Serve range only if validator still matches, else full 200 | `"9f2c-a1b2c3"` or date | [→](../13-Range-Requests/Range-Requests-Overview.md) |

**See also:** [Range Requests Overview](../13-Range-Requests/Range-Requests-Overview.md), [Content-Length](../04-Response-Headers/Content-Length.md) (length of the partial body).

---

## Proxy / Forwarding

| Header | Dir | Purpose | Typical value | Page |
|---|---|---|---|---|
| Forwarded | req | Standardized client/proxy chain metadata (RFC 7239) | `for=192.0.2.60;proto=https;by=203.0.113.43` | [→](../14-Proxies/Forwarded.md) |
| X-Forwarded-For | req | De-facto client IP chain through proxies | `203.0.113.7, 198.51.100.2` | [→](../14-Proxies/X-Forwarded-For.md) |
| X-Forwarded-Proto | req | Original scheme before TLS termination | `https` | [→](../14-Proxies/X-Forwarded-Proto.md) |
| Via | both | Proxies/gateways the message passed through | `1.1 vegur, 1.1 varnish` | [→](../14-Proxies/Via.md) |
| Connection | req/both | Hop-by-hop control; lists headers to strip per hop | `keep-alive` / `close` / `upgrade` | [→](../03-Request-Headers/Connection.md) |
| Host | req | Target host:port (routing on shared IPs) | `api.example.com` | [→](../03-Request-Headers/Host.md) |
| Upgrade | req | Request protocol switch (WebSocket, h2c) | `websocket` | [→](../03-Request-Headers/Upgrade.md) |

**See also:** [Proxies Overview](../14-Proxies/Proxies-Overview.md), [Reverse Proxy Overview](../16-Reverse-Proxies/Reverse-Proxy-Overview.md), [hop-by-hop headers](./Glossary.md#hop-by-hop-header), [Proxy-Authenticate / Proxy-Authorization](#authentication).

---

## Cookies

| Header | Dir | Purpose | Typical value | Page |
|---|---|---|---|---|
| Set-Cookie | res | Create/update/delete a cookie in the browser jar | `sid=abc; HttpOnly; Secure; SameSite=Lax; Path=/` | [→](../08-Cookies/Set-Cookie.md) |
| Cookie | req | Send stored cookies back to the origin | `sid=abc; theme=dark` | [→](../08-Cookies/Cookie.md) |

**See also:** [Cookies Overview](../08-Cookies/Cookies-Overview.md), [SameSite](./Glossary.md#samesite), [CHIPS](./Glossary.md#chips-partitioned-cookies), [Vary: Cookie](../06-Caching-Headers/Vary.md).

---

## Request-general

| Header | Dir | Purpose | Typical value | Page |
|---|---|---|---|---|
| Host | req | Target host:port; mandatory in HTTP/1.1 (`:authority` in h2/h3) | `www.example.com` | [→](../03-Request-Headers/Host.md) |
| User-Agent | req | Client software identification | `Mozilla/5.0 (…) Chrome/…` | [→](../03-Request-Headers/User-Agent.md) |
| Referer | req | URL of the page that initiated the request | `https://example.com/search?q=…` | [→](../03-Request-Headers/Referer.md) |
| Origin | req | Scheme+host+port of the initiator (CORS/CSRF) | `https://app.example.com` | [→](../03-Request-Headers/Origin.md) |
| Content-Type | req | Media type of the request body | `application/json` | [→](../03-Request-Headers/Content-Type.md) |
| Content-Length | req | Byte length of the request body | `348` | [→](../03-Request-Headers/Content-Length.md) |
| Connection | req | Hop-by-hop connection management | `keep-alive` | [→](../03-Request-Headers/Connection.md) |
| Expect | req | Ask server to confirm before sending body | `100-continue` | [→](../03-Request-Headers/Expect.md) |
| Upgrade | req | Request a protocol upgrade | `websocket` | [→](../03-Request-Headers/Upgrade.md) |
| Sec-Fetch-* | req | Browser-set request context (mode/site/dest/user) | `Sec-Fetch-Site: same-origin` | [→](../03-Request-Headers/Sec-Fetch.md) |

**See also:** [Request vs Response Headers](../01-Introduction/Request-vs-Response-Headers.md), [Accept](../03-Request-Headers/Accept.md), [Authorization](../09-Authentication/Authorization.md), [Cookie](../08-Cookies/Cookie.md).

---

## Response-general

| Header | Dir | Purpose | Typical value | Page |
|---|---|---|---|---|
| Content-Type | res | Media type (+ charset) of the response body | `text/html; charset=utf-8` | [→](../04-Response-Headers/Content-Type.md) |
| Content-Length | res | Byte length of the response body | `146515` | [→](../04-Response-Headers/Content-Length.md) |
| Content-Disposition | res | Inline vs attachment + download filename | `attachment; filename="report.pdf"` | [→](../04-Response-Headers/Content-Disposition.md) |
| Location | res | Redirect target (`3xx`) or created resource (`201`) | `https://example.com/login` | [→](../04-Response-Headers/Location.md) |
| Server | res | Origin software identification | `nginx/1.27.0` | [→](../04-Response-Headers/Server.md) |
| Date | res | Time the response was generated (age math) | `Tue, 07 Jul 2026 09:00:00 GMT` | [→](../04-Response-Headers/Date.md) |
| Retry-After | res | When to retry after `429`/`503` (or `3xx`) | `120` / `Wed, 21 Oct 2026 07:28:00 GMT` | [→](../04-Response-Headers/Retry-After.md) |
| Allow | res | Methods a resource supports (sent with `405`/`OPTIONS`) | `GET, POST, OPTIONS` | [→](../04-Response-Headers/Allow.md) |
| Accept-Ranges | res | Range support advertisement | `bytes` | [→](../04-Response-Headers/Accept-Ranges.md) |

**See also:** [Status Code → Header Map](./Status-Code-Header-Map.md), [Anatomy of an HTTP Message](../01-Introduction/Anatomy-of-an-HTTP-Message.md).

---

## Quick legend

- **Dir** — `req`: sent by client. `res`: sent by server. `both`: appears in either direction (often with different semantics — see the page).
- **Hop-by-hop** headers (`Connection`, `Transfer-Encoding`, `Upgrade`, `Proxy-Authenticate`, `Proxy-Authorization`, and anything listed in `Connection`) apply to a single connection and are stripped by proxies — never forwarded end-to-end. See [End-to-End vs Hop-by-Hop Headers](../01-Introduction/End-to-End-vs-Hop-by-Hop-Headers.md).
- **q-value** — a `;q=0.0–1.0` weight on negotiation headers (`Accept*`) expressing preference; default `1.0`.
- For the full ABNF grammar of any value, see the header's own page and [Header Syntax and Grammar](../02-Core-Concepts/Header-Syntax-and-Grammar.md).
</content>
</invoke>
