# Status Code → Header Map

> A status code is only half the message. Most codes are *only correct* when they arrive with specific headers — a `301` without [`Location`](../04-Response-Headers/Location.md) is broken, a `401` without [`WWW-Authenticate`](../09-Authentication/WWW-Authenticate.md) violates the spec, a `206` without [`Content-Range`](../13-Range-Requests/Content-Range.md) is unparseable. This table maps the codes you actually operate to the headers that must (or should) travel with them, and explains *why* each pairing exists.
>
> **Legend:** **MUST** = required by RFC 9110/9111; **SHOULD** = strongly expected; **often** = common in practice.

---

## Redirection (3xx)

| Code | Meaning | Companion header(s) | Why |
|---|---|---|---|
| 301 Moved Permanently | Resource has a new canonical URL | [`Location`](../04-Response-Headers/Location.md) **MUST** | Tells the client (and search engines) the new URL to follow and cache. Browsers may cache the redirect itself. |
| 302 Found | Temporary redirect | [`Location`](../04-Response-Headers/Location.md) **MUST** | The target for *this* request only; do not treat as canonical. Historically method-mangling (GET-ified). |
| 303 See Other | Redirect to a GET (Post/Redirect/Get) | [`Location`](../04-Response-Headers/Location.md) **MUST** | Forces the follow-up to be a `GET` regardless of original method — the safe PRG pattern after a form POST. |
| 307 Temporary Redirect | Temp redirect, method preserved | [`Location`](../04-Response-Headers/Location.md) **MUST** | Like 302 but the method/body are *preserved* (a POST stays a POST). |
| 308 Permanent Redirect | Permanent, method preserved | [`Location`](../04-Response-Headers/Location.md) **MUST** | Canonical move that keeps the method — the strict counterpart to 301. |

All redirects may also carry [`Cache-Control`](../06-Caching-Headers/Cache-Control.md) to control whether the redirect itself is cached, and [`Retry-After`](../04-Response-Headers/Retry-After.md) is permitted on `3xx` to hint when the target will be ready.

---

## Conditional & caching (304, 412, 428)

| Code | Meaning | Companion header(s) | Why |
|---|---|---|---|
| 304 Not Modified | Cached copy is still valid | [`ETag`](../06-Caching-Headers/ETag.md), [`Last-Modified`](../06-Caching-Headers/Last-Modified.md), [`Cache-Control`](../06-Caching-Headers/Cache-Control.md), [`Vary`](../06-Caching-Headers/Vary.md), [`Date`](../04-Response-Headers/Date.md) | Sent in reply to [`If-None-Match`](../12-Conditional-Requests/If-None-Match.md) / [`If-Modified-Since`](../12-Conditional-Requests/If-Modified-Since.md). Carries **no body**; must echo the validators and any caching/`Vary` headers so the cache can refresh its metadata and re-arm freshness. Sending a body on `304` is a protocol error. |
| 412 Precondition Failed | A conditional precondition failed | (echo current [`ETag`](../06-Caching-Headers/ETag.md)) | Result of `If-Match` / `If-Unmodified-Since` not matching — the optimistic-lock "someone else changed it" signal for writes. |
| 428 Precondition Required | Origin demands a conditional request | — | Forces clients to send `If-Match`/`If-None-Match` so unconditional writes can't clobber concurrent edits. |

---

## Authentication (401, 403, 407)

| Code | Meaning | Companion header(s) | Why |
|---|---|---|---|
| 401 Unauthorized | Missing/invalid credentials | [`WWW-Authenticate`](../09-Authentication/WWW-Authenticate.md) **MUST** | The challenge: names the scheme(s) and `realm` the client must satisfy (`Bearer`, `Basic`, `Digest`). Without it a `401` is malformed — the client has no idea *how* to authenticate. May include `error="invalid_token"` for OAuth. Client answers with [`Authorization`](../09-Authentication/Authorization.md). |
| 403 Forbidden | Authenticated but not allowed | (usually none) | No challenge — retrying with credentials won't help. Deliberately *omits* `WWW-Authenticate`; that distinction (401 = "who are you?" vs 403 = "not you") is the whole point. |
| 407 Proxy Authentication Required | Proxy needs credentials | `Proxy-Authenticate` **MUST** | The proxy's analogue of 401. Client answers with `Proxy-Authorization` (hop-by-hop, not forwarded upstream). See [Proxies Overview](../14-Proxies/Proxies-Overview.md). |

---

## Method & resource constraints (405, 406, 413, 415, 431)

| Code | Meaning | Companion header(s) | Why |
|---|---|---|---|
| 405 Method Not Allowed | Method unsupported on this URL | [`Allow`](../04-Response-Headers/Allow.md) **MUST** | Lists the methods the resource *does* support (e.g. `Allow: GET, POST, OPTIONS`) so the client can correct. Also returned on `OPTIONS`. |
| 406 Not Acceptable | Can't satisfy negotiation | [`Vary`](../06-Caching-Headers/Vary.md), sometimes an alternatives list | The server can't meet the [`Accept`](../03-Request-Headers/Accept.md) / [`Accept-Language`](../03-Request-Headers/Accept-Language.md) / `Accept-Encoding` constraints. |
| 413 Content Too Large | Body exceeds limit | [`Retry-After`](../04-Response-Headers/Retry-After.md) (if temporary) | Payload too big; `Retry-After` only if the limit is transient. |
| 415 Unsupported Media Type | Body's `Content-Type` unsupported | [`Accept-Post`](../11-Content-Negotiation/Content-Negotiation-Overview.md) / `Accept` (advisory) | Server rejects the request [`Content-Type`](../03-Request-Headers/Content-Type.md); may advertise what it *does* accept. |
| 431 Request Header Fields Too Large | Headers exceed server limits | (message body explaining which) | Total header size (or one oversized field — often a bloated [`Cookie`](../08-Cookies/Cookie.md) or `Referer`) exceeded the server/proxy cap. No standard companion header; the fix is smaller headers. See [Header Size Limits](../02-Core-Concepts/Header-Size-Limits.md). |

---

## Range requests (206, 416)

| Code | Meaning | Companion header(s) | Why |
|---|---|---|---|
| 206 Partial Content | Serving a byte range | [`Content-Range`](../13-Range-Requests/Content-Range.md) **MUST**, [`Content-Length`](../04-Response-Headers/Content-Length.md), [`Accept-Ranges`](../04-Response-Headers/Accept-Ranges.md) | Reply to a [`Range`](../13-Range-Requests/Range.md) request. `Content-Range: bytes 0-1023/146515` states which bytes and the total size; `Content-Length` is the length of *this chunk*. Multipart ranges use `Content-Type: multipart/byteranges`. Powers video seeking and resumable downloads. |
| 416 Range Not Satisfiable | Requested range invalid | [`Content-Range`](../13-Range-Requests/Content-Range.md) **MUST** (`bytes */<len>`) | The range fell outside the resource; `Content-Range: bytes */146515` tells the client the real total so it can retry with a valid range. |

A `200 OK` with [`Accept-Ranges: bytes`](../04-Response-Headers/Accept-Ranges.md) is how the server *advertises* range support in the first place; `Accept-Ranges: none` refuses it.

---

## Rate limiting & availability (429, 503)

| Code | Meaning | Companion header(s) | Why |
|---|---|---|---|
| 429 Too Many Requests | Client is rate-limited | [`Retry-After`](../04-Response-Headers/Retry-After.md) **SHOULD**, `RateLimit-*` | `Retry-After` (seconds or HTTP-date) tells a well-behaved client when to back off — essential for automated retries. Often paired with draft `RateLimit-Limit`/`-Remaining`/`-Reset`. |
| 503 Service Unavailable | Temporarily down/overloaded | [`Retry-After`](../04-Response-Headers/Retry-After.md) **SHOULD** | Signals maintenance or overload and when to come back; load balancers and CDNs read it to schedule retries and serve `stale-if-error`. |

---

## Success & informational (100, 201, 200)

| Code | Meaning | Companion header(s) | Why |
|---|---|---|---|
| 100 Continue | OK to send the request body | (triggered by [`Expect: 100-continue`](../03-Request-Headers/Expect.md)) | The client sent `Expect: 100-continue` and paused; this interim response says "headers accepted, send the body." Lets a server reject a huge upload (on auth/size) *before* the body is transmitted. |
| 101 Switching Protocols | Protocol upgrade accepted | [`Upgrade`](../03-Request-Headers/Upgrade.md), [`Connection: Upgrade`](../03-Request-Headers/Connection.md) | Confirms a switch requested via `Upgrade` (WebSocket handshake, HTTP/1.1→h2c). |
| 200 OK | Success | [`Content-Type`](../04-Response-Headers/Content-Type.md), [`Content-Length`](../04-Response-Headers/Content-Length.md) / `Transfer-Encoding`, [`Cache-Control`](../06-Caching-Headers/Cache-Control.md), [`ETag`](../06-Caching-Headers/ETag.md) | The baseline: describe the body, its length/framing, and how it may be cached and revalidated. |
| 201 Created | Resource created | [`Location`](../04-Response-Headers/Location.md) **SHOULD**, `Content-Location` | `Location` points at the newly created resource's URL (e.g. after a `POST`). |
| 204 No Content | Success, empty body | (no `Content-Length` body; may carry [`ETag`](../06-Caching-Headers/ETag.md)) | Used by CORS preflight replies and successful writes with nothing to return. |

---

## CORS preflight (204 / 200 to OPTIONS)

A preflight `OPTIONS` response (usually `204` or `200`) is defined almost entirely by its headers, not its code:

| Header | Role |
|---|---|
| [`Access-Control-Allow-Origin`](../07-CORS/Access-Control-Allow-Origin.md) | Which origin may proceed |
| [`Access-Control-Allow-Methods`](../07-CORS/Access-Control-Allow-Methods.md) | Answers `Access-Control-Request-Method` |
| [`Access-Control-Allow-Headers`](../07-CORS/Access-Control-Allow-Headers.md) | Answers `Access-Control-Request-Headers` |
| [`Access-Control-Allow-Credentials`](../07-CORS/Access-Control-Allow-Credentials.md) | Permit cookies/credentials |
| [`Access-Control-Max-Age`](../07-CORS/Access-Control-Max-Age.md) | How long to cache this preflight |

Add [`Vary: Origin`](../06-Caching-Headers/Vary.md) whenever `Access-Control-Allow-Origin` reflects the request origin, or a shared cache will serve one origin's ACAO to another. See [CORS Overview](../07-CORS/CORS-Overview.md).

---

## One-line memory hooks

- **3xx → `Location`** (where to go)
- **304 → validators + `Cache-Control` + `Vary`** (refresh metadata, no body)
- **401 → `WWW-Authenticate`** / **407 → `Proxy-Authenticate`** (how to authenticate)
- **405 → `Allow`** (which methods instead)
- **206 → `Content-Range` + `Accept-Ranges`** / **416 → `Content-Range: */len`**
- **429 / 503 → `Retry-After`** (when to come back)
- **100 ← `Expect: 100-continue`** (may I send the body?)
- **431 → shrink your headers** (no companion; usually a bloated `Cookie`)

See also: [Status codes glossary entry](./Glossary.md#status-code), [Location](../04-Response-Headers/Location.md), [Retry-After](../04-Response-Headers/Retry-After.md), [Anatomy of an HTTP Message](../01-Introduction/Anatomy-of-an-HTTP-Message.md).
</content>
