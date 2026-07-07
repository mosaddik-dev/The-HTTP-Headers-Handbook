# Cloudflare-Specific Headers

> A **chapter page** cataloguing the non-standard, vendor-specific headers Cloudflare adds to requests and responses. These aren't part of any RFC — they're Cloudflare's operational vocabulary — but you *will* encounter them constantly when your app sits behind Cloudflare, and knowing them is essential for debugging, correct client-IP recovery, and cache/security tuning. Where a Cloudflare header duplicates a standard one, this page points you to the standard (e.g. prefer [`X-Forwarded-For`](../14-Proxies/X-Forwarded-For.md)/[`Forwarded`](../14-Proxies/Forwarded.md) semantics, but use `CF-Connecting-IP` for the authoritative client IP behind Cloudflare).

## Why vendor headers exist

Cloudflare (like every large CDN) sits between your users and your origin as a reverse proxy, terminating TLS, caching, filtering, and routing. In doing so it (a) **hides** facts your origin needs (the real client IP, the client's country, the TLS details) and (b) **learns** operational facts worth surfacing (cache hit/miss, which datacenter served the request, bot scores). Standard headers cover some of this ([`X-Forwarded-For`](../14-Proxies/X-Forwarded-For.md), [`Via`](../14-Proxies/Via.md), [`Age`](../06-Caching-Headers/Age.md)), but Cloudflare adds its own, more authoritative and richer, set. The two most important reasons you care:

1. **Client-IP recovery** behind Cloudflare should use **`CF-Connecting-IP`** (Cloudflare's authoritative view) rather than parsing a possibly-multi-entry [`X-Forwarded-For`](../14-Proxies/X-Forwarded-For.md).
2. **Debugging** cache and delivery requires Cloudflare's `cf-cache-status`, `cf-ray`, and friends.

## Request headers Cloudflare adds (origin sees these)

These appear on requests **from Cloudflare to your origin**:

| Header | Meaning | Standard equivalent / notes |
|---|---|---|
| **`CF-Connecting-IP`** | The **real client IP** (authoritative). | Prefer over [`X-Forwarded-For`](../14-Proxies/X-Forwarded-For.md) behind Cloudflare. IPv4/IPv6. |
| **`CF-Connecting-IPv6`** | Client IPv6 (in pseudo-IPv4 setups). | Situational. |
| **`True-Client-IP`** | Same as `CF-Connecting-IP` (Enterprise). | Matches Akamai's convention. |
| **`CF-IPCountry`** | Two-letter country code of the client (geo). | e.g. `US`, `DE`, `T1` (Tor), `XX` (unknown). |
| **`CF-Ray`** | Unique request ID (also on responses). | Your primary correlation ID for support/logs. |
| **`CF-Visitor`** | JSON with the client's scheme, e.g. `{"scheme":"https"}`. | Complements [`X-Forwarded-Proto`](../14-Proxies/X-Forwarded-Proto.md). |
| **`CF-Worker`** | Set when a Cloudflare Worker made a subrequest. | Loop/subrequest detection. |
| **`CDN-Loop`** | Standard-ish loop-prevention token (`CDN-Loop: cloudflare`). | RFC 8586; prevents CDN loops. |
| **`CF-EW-Via`** | Edge Worker routing info. | Internal-ish. |
| `X-Forwarded-For` / `X-Forwarded-Proto` | Also set/appended by Cloudflare. | Standard; but `CF-Connecting-IP`/`CF-Visitor` are more authoritative here. |

## Response headers Cloudflare adds (client sees these)

These appear on responses **from Cloudflare to the browser**:

| Header | Meaning |
|---|---|
| **`CF-Cache-Status`** | Cache outcome: `HIT`, `MISS`, `EXPIRED`, `STALE`, `REVALIDATED`, `UPDATING`, `BYPASS`, `DYNAMIC` (not cached). Your #1 cache-debugging signal. |
| **`CF-Ray`** | The request's unique ID + datacenter code (e.g. `7d3f...-FRA`). Correlates client ↔ Cloudflare ↔ support. |
| **`Server: cloudflare`** | Identifies Cloudflare as the serving edge (overwrites origin [`Server`](../04-Response-Headers/Server.md)). |
| **`CF-Cache-Status: DYNAMIC`** | Explicitly "not cached" (default for uncacheable/dynamic content). |
| `Age` | Standard [`Age`](../06-Caching-Headers/Age.md) — how long the edge held the object. |
| `cf-apo-via` | Automatic Platform Optimization info (WordPress etc.). |
| `NEL` / `Report-To` | Cloudflare can inject [`NEL`](../05-Security-Headers/NEL.md)/[`Report-To`](../05-Security-Headers/Reporting-Endpoints-Report-To.md) for its own monitoring. |

## How it works (request flow)

```mermaid
sequenceDiagram
    participant B as Browser (203.0.113.7)
    participant CF as Cloudflare edge (FRA)
    participant O as Origin
    B->>CF: HTTPS GET /
    Note over CF: Add CF-Connecting-IP, CF-IPCountry,<br/>CF-Ray, CF-Visitor, CDN-Loop
    CF->>O: GET /<br/>CF-Connecting-IP: 203.0.113.7<br/>CF-IPCountry: DE<br/>CF-Ray: 7d3f...-FRA<br/>CF-Visitor: {"scheme":"https"}<br/>X-Forwarded-For: 203.0.113.7
    O-->>CF: 200 OK (Cache-Control: public, max-age=3600)
    Note over CF: Cache; add CF-Cache-Status, CF-Ray, Server: cloudflare
    CF-->>B: 200 OK<br/>CF-Cache-Status: MISS<br/>CF-Ray: 7d3f...-FRA<br/>Age: 0
```

## Express.js Example — recovering client IP and geo behind Cloudflare

```js
const express = require('express');
const app = express();

// Trust Cloudflare's edge, then prefer its authoritative headers.
app.set('trust proxy', true); // set precisely in prod; ideally restrict to Cloudflare IP ranges.

app.use((req, res, next) => {
  // 1) Client IP: CF-Connecting-IP is authoritative behind Cloudflare —
  //    more reliable than parsing X-Forwarded-For.
  req.clientIp = req.headers['cf-connecting-ip']
    || req.headers['true-client-ip']            // Enterprise
    || req.ip;                                  // fallback (XFF-derived)

  // 2) Geo without a GeoIP DB — Cloudflare provides the country.
  req.country = req.headers['cf-ipcountry'] || 'XX';

  // 3) Correlation ID for logs/support tickets.
  req.rayId = req.headers['cf-ray'];

  // 4) Client scheme (complements X-Forwarded-Proto).
  try {
    req.clientScheme = JSON.parse(req.headers['cf-visitor'] || '{}').scheme || req.protocol;
  } catch { req.clientScheme = req.protocol; }

  next();
});

app.get('/whoami', (req, res) => {
  res.json({ ip: req.clientIp, country: req.country, ray: req.rayId, scheme: req.clientScheme });
});

app.listen(3000);
```

Why this matters: behind Cloudflare, `req.socket.remoteAddress` is a **Cloudflare edge IP**, and [`X-Forwarded-For`](../14-Proxies/X-Forwarded-For.md) may contain multiple entries — so `CF-Connecting-IP` is the **single, authoritative client IP** and the recommended source. `CF-IPCountry` gives you free geolocation without a MaxMind database. `CF-Ray` is invaluable: log it, and when a user reports an issue you can match their `CF-Ray` (visible in their response headers) to your logs and Cloudflare's. **Critically**, only trust these headers from **Cloudflare's IP ranges** — otherwise an attacker hitting your origin directly could forge `CF-Connecting-IP` (see Security).

## Security: lock the origin to Cloudflare

The single most important security practice: **all these headers are forgeable if an attacker can reach your origin directly**. `CF-Connecting-IP: 127.0.0.1` sent straight to your origin would spoof the client IP unless you restrict trust.

- **Firewall the origin** to accept connections **only from [Cloudflare's published IP ranges](https://www.cloudflare.com/ips/)** (or use Cloudflare Tunnel / Authenticated Origin Pulls / mTLS). Then `CF-Connecting-IP` etc. are trustworthy.
- **Never trust `CF-*` headers** if the request could have bypassed Cloudflare.
- Combine with [`X-Forwarded-For`](../14-Proxies/X-Forwarded-For.md) trust discipline (`trust proxy` scoped to Cloudflare ranges).
- Treat `CF-Connecting-IP` (client IP) as **PII** (GDPR).
- `CDN-Loop: cloudflare` helps prevent request loops through Cloudflare; don't strip it upstream inappropriately.

## Cache debugging with `CF-Cache-Status`

`CF-Cache-Status` is your primary cache oracle:

- **`HIT`** — served from edge cache. 🎉
- **`MISS`** — not in cache; fetched from origin (now cached if cacheable).
- **`EXPIRED`** — was cached but stale; revalidated/refetched.
- **`REVALIDATED`** — stale but confirmed unchanged via [`ETag`](../06-Caching-Headers/ETag.md)/[`If-None-Match`](../12-Conditional-Requests/If-None-Match.md).
- **`STALE`** — served stale (e.g. `stale-while-revalidate` / origin down).
- **`UPDATING`** — served stale while refreshing.
- **`BYPASS`** — caching bypassed (e.g. a bypass rule, `Cache-Control: private`).
- **`DYNAMIC`** — not eligible for caching (default for many content types unless you configure caching).

Pair it with [`Age`](../06-Caching-Headers/Age.md) (how long held) and standard cache headers to diagnose "why isn't this cached?" (often `DYNAMIC` because Cloudflare by default only caches static file extensions unless you add a **Cache Rule** / `Cache-Control` that opts HTML/API responses in).

## React Example — reading Cloudflare context in the browser

```jsx
// CF-Ray and CF-Cache-Status are on the RESPONSE; JS can read them same-origin
// (or cross-origin only if exposed via Access-Control-Expose-Headers).
async function fetchWithDiagnostics(url) {
  const res = await fetch(url);
  const diagnostics = {
    ray: res.headers.get('cf-ray'),               // include in error reports/support tickets
    cache: res.headers.get('cf-cache-status'),    // HIT/MISS/etc.
    age: res.headers.get('age'),
  };
  return { data: await res.json(), diagnostics };
}
// Surfacing CF-Ray in your error UI lets users quote it to support for fast tracing.
```

Note: to read `CF-*` response headers from **cross-origin** JS, the response must include [`Access-Control-Expose-Headers`](../07-CORS/Access-Control-Expose-Headers.md).

## Common Mistakes

- **Trusting `CF-*` headers without locking the origin to Cloudflare IPs** → spoofable client IP/geo. Firewall the origin.
- **Parsing [`X-Forwarded-For`](../14-Proxies/X-Forwarded-For.md) instead of using `CF-Connecting-IP`** behind Cloudflare — the latter is authoritative and simpler.
- **Expecting HTML/API to be cached by default** — Cloudflare defaults to `DYNAMIC` (no cache) for many types; you must add Cache Rules / correct `Cache-Control`.
- **Ignoring `CF-Ray`** — it's the fastest way to correlate an incident across client, Cloudflare, and origin.
- **Stripping `CDN-Loop`** or mishandling loop-prevention.
- **Assuming `CF-*` are standard** — they're vendor-specific; other CDNs use different names ([see CDN Debugging Headers](./CDN-Debugging-Headers.md)).
- **Not exposing `CF-*` via CORS** when a cross-origin browser app needs to read them.

## Best Practices

- [ ] **Firewall the origin to Cloudflare IP ranges** (or use Tunnel/Authenticated Origin Pulls) so `CF-*` headers are trustworthy.
- [ ] Use **`CF-Connecting-IP`** (or `True-Client-IP` on Enterprise) as the authoritative client IP; fall back to [`X-Forwarded-For`](../14-Proxies/X-Forwarded-For.md).
- [ ] **Log `CF-Ray`** on every request for cross-system correlation.
- [ ] Use **`CF-IPCountry`** for geo instead of a separate GeoIP lookup where sufficient.
- [ ] Watch **`CF-Cache-Status`** + [`Age`](../06-Caching-Headers/Age.md) to verify/tune caching; add Cache Rules to cache HTML/API deliberately.
- [ ] Treat client IPs as **PII**; secure and retain per policy.
- [ ] Expose needed `CF-*` headers via [`Access-Control-Expose-Headers`](../07-CORS/Access-Control-Expose-Headers.md) for cross-origin browser apps.
- [ ] Don't hard-code Cloudflare specifics into portable code — abstract client-IP/geo behind a helper so switching CDNs is easy.

## Related Pages

- [X-Forwarded-For](../14-Proxies/X-Forwarded-For.md) / [Forwarded](../14-Proxies/Forwarded.md) — standard client-IP forwarding; `CF-Connecting-IP` is Cloudflare's authoritative version.
- [X-Forwarded-Proto](../14-Proxies/X-Forwarded-Proto.md) — standard scheme; complements `CF-Visitor`.
- [Via](../14-Proxies/Via.md) — standard proxy-chain header.
- [Age](../06-Caching-Headers/Age.md) — standard cache-age; pair with `CF-Cache-Status`.
- [CDN Debugging Headers](./CDN-Debugging-Headers.md) — the vendor-neutral catalogue (Fastly/CloudFront/Akamai equivalents).
- [CDN Caching Overview](./CDN-Caching-Overview.md) / [Cache Keys and Vary](./Cache-Keys-and-Vary.md) — the caching model.
- [Access-Control-Expose-Headers](../07-CORS/Access-Control-Expose-Headers.md) — to read `CF-*` cross-origin.

## Mental Model

Think of Cloudflare's headers as the **stamps and annotations a global mail-forwarding service adds to your parcels.** When mail passes through their sorting hub, they staple on notes your receiving office genuinely needs — "*actual sender's address*" (`CF-Connecting-IP`), "*posted from Germany*" (`CF-IPCountry`), a "*tracking number, sorted at Frankfurt hub*" (`CF-Ray`) — and, on the way back to the sender, "*served from our local shelf / had to fetch from the warehouse*" (`CF-Cache-Status`). These annotations are enormously useful, but there's a catch that governs everything: the notes are only trustworthy **if the parcel truly came through their hub**. If you accept parcels dropped directly at your back door (origin reachable bypassing Cloudflare), a fraudster can pre-scribble "actual sender: the police" and you'd believe it. So the golden rule is: **only accept mail through the forwarding hub's own trucks** (firewall the origin to Cloudflare IPs) — then the stamps are gold.
