# HTTP Header Anti-Patterns

> A **best-practices chapter** cataloguing the most common, most damaging header mistakes — grouped by category — with *why it's wrong*, *what breaks*, and *the fix*. Each entry links to the relevant deep-dive. Treat this as a pre-deploy review checklist: if your responses exhibit any of these, fix them. The anti-patterns here are drawn from the pitfalls documented throughout the handbook, concentrated in one place.

## Security anti-patterns

### ❌ `Access-Control-Allow-Origin: *` with `Access-Control-Allow-Credentials: true`
**Why it's wrong:** illegal per spec — the browser blocks it. **What breaks:** all credentialed cross-origin requests fail. **Worse:** reflecting *any* origin (`res.set('ACAO', req.headers.origin)`) with credentials enabled is a **data-theft hole** letting any site read authenticated responses.
**Fix:** allowlist origins; echo a *specific* [`Access-Control-Allow-Origin`](../07-CORS/Access-Control-Allow-Origin.md) + [`ACAC: true`](../07-CORS/Access-Control-Allow-Credentials.md) + [`Vary: Origin`](../06-Caching-Headers/Vary.md).

### ❌ Treating CORS as CSRF protection
**Why it's wrong:** CORS controls whether JS can *read* a response; the request still *executes* server-side with cookies attached. **What breaks:** state-changing requests (transfers, deletes) succeed even when JS can't read the reply → CSRF.
**Fix:** [`SameSite=Lax/Strict`](../08-Cookies/Set-Cookie.md) cookies + CSRF tokens + [`Origin`](../03-Request-Headers/Origin.md) validation for unsafe methods. See [Auth Architecture](../20-Real-World-Architectures/Auth-Architecture-End-to-End.md).

### ❌ Session cookies without the full flag stack
**Why it's wrong:** a session cookie missing `HttpOnly` is stealable via XSS; missing `Secure` leaks over HTTP; missing [`SameSite`](../08-Cookies/Set-Cookie.md) is CSRF-prone.
**Fix:** `Set-Cookie: sid=...; HttpOnly; Secure; SameSite=Lax; Path=/` (ideally `__Host-` prefix), over HTTPS with [HSTS](../05-Security-Headers/Strict-Transport-Security.md).

### ❌ `trust proxy: true` (trust all forwarding headers)
**Why it's wrong:** any client can forge [`X-Forwarded-For`](../14-Proxies/X-Forwarded-For.md)/[`-Proto`](../14-Proxies/X-Forwarded-Proto.md)/[`-Host`](../14-Proxies/X-Forwarded-Host.md). **What breaks:** IP spoofing (rate-limit/allowlist bypass), forged HTTPS state (insecure `Secure` cookies), host-header injection.
**Fix:** set `trust proxy` to the **exact** hop count/subnets; lock the origin to the CDN/proxy IPs.

### ❌ Reflecting `X-Forwarded-Host` into URLs without validation
**Why it's wrong:** enables **host-header injection** → poisoned password-reset links (account takeover), open redirects, cache poisoning.
**Fix:** allowlist hostnames; build links only from a validated host. See [X-Forwarded-Host](../14-Proxies/X-Forwarded-Host.md).

### ❌ Leaving `TRACE` enabled / verbose `Server` & `Via`
**Why it's wrong:** `TRACE` enables [Cross-Site Tracing](../03-Request-Headers/Max-Forwards.md) (credential exfiltration); verbose [`Server`](../04-Response-Headers/Server.md)/[`Via`](../14-Proxies/Via.md) leak software versions for CVE targeting.
**Fix:** disable `TRACE` (405); strip/pseudonymize version-revealing headers at the edge.

### ❌ Missing security headers baseline
**Why it's wrong:** no [CSP](../05-Security-Headers/Content-Security-Policy.md) (XSS), no [HSTS](../05-Security-Headers/Strict-Transport-Security.md) (downgrade), no [`X-Content-Type-Options: nosniff`](../05-Security-Headers/X-Content-Type-Options.md) (MIME sniffing), no [`X-Frame-Options`](../05-Security-Headers/X-Frame-Options.md)/frame-ancestors (clickjacking).
**Fix:** apply a baseline (helmet or equivalent) — see [Security Hardening](./Security-Hardening.md).

## Caching anti-patterns

### ❌ Caching authenticated content in shared caches
**Why it's wrong:** a CDN/proxy caches a `private` response and serves **user A's data to user B**. The most damaging caching bug.
**Fix:** `Cache-Control: private, no-store` on all authenticated/personalized responses. See [Caching Architecture](../20-Real-World-Architectures/Caching-Architecture-End-to-End.md).

### ❌ Fleet-inconsistent `ETag`s (mtime/inode-based)
**Why it's wrong:** each server emits a different [`ETag`](../06-Caching-Headers/ETag.md) for identical bytes → [`If-None-Match`](../12-Conditional-Requests/If-None-Match.md) never matches → `304`s never happen → full re-downloads.
**Fix:** content-hash or shared-version ETags, consistent across the fleet.

### ❌ Long-caching un-versioned assets
**Why it's wrong:** `Cache-Control: max-age=31536000` on `/app.js` (no hash) → users stuck with stale JS after deploy.
**Fix:** content-hash filenames (`/app.9f2c.js`) + `immutable`; or short TTL + revalidation for un-hashed. See [Caching Strategy](./Caching-Strategy.md).

### ❌ Missing `Vary: Accept-Encoding`
**Why it's wrong:** a shared cache serves a **Brotli body to a gzip-only client** → binary garbage.
**Fix:** [`Vary: Accept-Encoding`](../06-Caching-Headers/Vary.md) on compressible responses; let compression middleware set it and don't clobber it.

### ❌ `Vary: User-Agent` or `Vary: Cookie`
**Why it's wrong:** near-infinite cardinality → cache fragmentation collapse (hit ratio ≈ 0); `Vary: Cookie` can also leak private data.
**Fix:** normalize to low-cardinality derived headers; use `private` for per-user content.

### ❌ Relying on `Pragma: no-cache` / `Expires: 0` on responses
**Why it's wrong:** response [`Pragma`](../06-Caching-Headers/Pragma.md) has **no standardized effect**; sensitive data may still be cached.
**Fix:** use [`Cache-Control: no-store`](../06-Caching-Headers/Cache-Control.md).

### ❌ Aggressive TTLs with no purge mechanism
**Why it's wrong:** content changes (or a security fix ships) but stale copies serve until TTL expiry.
**Fix:** event-driven purge (URL or [tag](../06-Caching-Headers/Surrogate-Control.md)) on change; active purge after security fixes.

## Framing / protocol anti-patterns

### ❌ Sending both `Content-Length` and `Transfer-Encoding`
**Why it's wrong:** ambiguous framing → **HTTP request smuggling** if two hops disagree.
**Fix:** never emit both; let the runtime pick one. See [Content-Length vs Transfer-Encoding](../10-Compression/Content-Length-vs-Transfer-Encoding.md).

### ❌ `Transfer-Encoding: chunked` / `Connection` / `Keep-Alive` on HTTP/2 or /3
**Why it's wrong:** these are HTTP/1.1 hop-by-hop headers; they're **protocol errors** in h2/h3.
**Fix:** don't set connection-management headers on h2/h3; the protocol handles framing/multiplexing.

### ❌ Forwarding hop-by-hop headers across hops
**Why it's wrong:** [`Connection`](../03-Request-Headers/Connection.md), [`Keep-Alive`](../04-Response-Headers/Keep-Alive.md), [`TE`](../03-Request-Headers/TE.md), [`Proxy-Authorization`](../09-Authentication/Proxy-Authorization.md) are per-hop; forwarding them leaks credentials or corrupts connection handling.
**Fix:** strip hop-by-hop headers at each proxy. See [End-to-End vs Hop-by-Hop](../01-Introduction/End-to-End-vs-Hop-by-Hop-Headers.md).

### ❌ Server keep-alive shorter than the LB idle timeout
**Why it's wrong:** the LB reuses a socket the app just closed → intermittent `502`s.
**Fix:** set the app's [`keepAliveTimeout`](../04-Response-Headers/Keep-Alive.md) **above** the LB's; keep Node's `headersTimeout > keepAliveTimeout`.

## Content / correctness anti-patterns

### ❌ Missing or wrong `Content-Type` / charset
**Why it's wrong:** MIME sniffing (security risk), mojibake (missing `charset=utf-8`), broken rendering.
**Fix:** always set an accurate [`Content-Type`](../04-Response-Headers/Content-Type.md) with `charset=utf-8` for text; add [`X-Content-Type-Options: nosniff`](../05-Security-Headers/X-Content-Type-Options.md).

### ❌ Confusing `Location` with `Content-Location`
**Why it's wrong:** [`Location`](../04-Response-Headers/Location.md) navigates (redirects); [`Content-Location`](../04-Response-Headers/Content-Location.md) is metadata. Swapping breaks redirects or misleads clients.
**Fix:** use `Location` for redirects/created resources; `Content-Location` only to name a returned representation's URL.

### ❌ Sending a body with `304` / `204`
**Why it's wrong:** protocol violation; confuses caches/clients.
**Fix:** `res.status(304).end()` / `res.status(204).end()` — no body.

### ❌ Implementing charset negotiation via `Accept-Charset`
**Why it's wrong:** obsolete; browsers don't send it. Dead code.
**Fix:** serve UTF-8, declare it; ignore [`Accept-Charset`](../03-Request-Headers/Accept-Charset.md).

### ❌ Omitting `Allow` on `405`
**Why it's wrong:** spec-required; a bare `405` leaves clients guessing valid methods.
**Fix:** [`Allow: GET, POST, ...`](../04-Response-Headers/Allow.md) on every `405`.

## Reliability / operational anti-patterns

### ❌ `429`/`503` without `Retry-After`
**Why it's wrong:** clients retry immediately → thundering-herd amplifies the outage.
**Fix:** [`Retry-After`](../04-Response-Headers/Retry-After.md) with an accurate delay; clients honor it with jitter.

### ❌ Large JWTs/cookies in headers
**Why it's wrong:** exceed [header-size limits](../02-Core-Concepts/Header-Size-Limits.md) at a proxy/CDN → `431`/`400` for affected users.
**Fix:** short opaque session IDs + server-side state; cookie hygiene; lean tokens.

### ❌ Missing client-IP recovery behind a CDN
**Why it's wrong:** logs/rate-limits key on the CDN's IP (all users look like one); or naive leftmost-[`X-Forwarded-For`](../14-Proxies/X-Forwarded-For.md) trust enables spoofing.
**Fix:** derive client IP from trusted forwarding headers (or [`CF-Connecting-IP`](../15-CDNs/Cloudflare-Specific-Headers.md)) with correct trust config.

### ❌ No observability for policy rollouts
**Why it's wrong:** deploying strict [CSP](../05-Security-Headers/Content-Security-Policy.md)/[COEP](../05-Security-Headers/Cross-Origin-Embedder-Policy.md) blind breaks the site.
**Fix:** roll out **report-only** first via [`Reporting-Endpoints`](../05-Security-Headers/Reporting-Endpoints-Report-To.md); watch reports; then enforce.

## Quick self-audit table

| Symptom | Likely anti-pattern | Fix |
|---|---|---|
| One user sees another's data | Auth response shared-cached | `private, no-store` |
| Compressed page = garbage | Missing `Vary: Accept-Encoding` | Add it |
| CORS error with credentials | `ACAO: *` + credentials | Specific origin + `Vary: Origin` |
| Stale JS after deploy | Un-versioned long cache | Content-hash + `immutable` |
| Intermittent `502` behind LB | keep-alive race | Raise app keep-alive above LB |
| `431`/`400` for some users | Header bloat | Shrink cookies/tokens |
| Reset link points to attacker | XFH reflected unvalidated | Allowlist hosts |
| No `304`s, high origin load | Fleet-inconsistent ETags | Content-hash ETags |
| Password reset works via curl not browser | CORS/CSRF confusion | Fix CORS + keep CSRF defense |

## Related Pages

- [Security Hardening](./Security-Hardening.md) — the positive security checklist.
- [Caching Strategy](./Caching-Strategy.md) — the positive caching checklist.
- [Auth Architecture End-to-End](../20-Real-World-Architectures/Auth-Architecture-End-to-End.md) / [Caching Architecture End-to-End](../20-Real-World-Architectures/Caching-Architecture-End-to-End.md) — the full flows.
- [Content-Length vs Transfer-Encoding](../10-Compression/Content-Length-vs-Transfer-Encoding.md) — smuggling.
- [Header Size Limits](../02-Core-Concepts/Header-Size-Limits.md) — `431`/`400` bloat.
- [Cheat Sheet](../22-Reference/Cheat-Sheet.md) — quick reference by category.

## Mental Model

Think of these anti-patterns as the **recurring failure modes an experienced code reviewer flags on sight** — the header equivalents of "off-by-one" or "SQL injection." Most of them share one root cause: **treating a header as decorative when it's load-bearing, or as trusted when it's forgeable.** `Vary` looks optional until a Brotli body renders as garbage; `trust proxy` looks like a checkbox until it's the difference between correct rate-limiting and an open spoofing hole; `Cache-Control: private` looks pedantic until a CDN serves one customer's bank balance to another. The cure is a single habit: for every header you set or trust, ask *"which tier consumes this, what happens if it's wrong, and can an attacker forge it?"* — the same three questions that turn a pile of headers into a correct, secure system.
