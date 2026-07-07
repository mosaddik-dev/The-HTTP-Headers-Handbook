# If-Modified-Since

## Quick Summary

`If-Modified-Since` is a **request** header carrying an HTTP date, used to make a `GET`/`HEAD` **conditional** on time: "only send me the full body if the resource has changed **since** this date; otherwise reply `304 Not Modified`." It is the date-based revalidation header, paired with the response's [`Last-Modified`](../06-Caching-Headers/Last-Modified.md) — the browser stores the `Last-Modified` timestamp of a cached copy and echoes it back in `If-Modified-Since` when the copy goes stale. It is the older, weaker sibling of [`If-None-Match`](./If-None-Match.md) (which uses opaque [`ETag`](../06-Caching-Headers/ETag.md) validators): when **both** are present on a request, the server evaluates `If-None-Match` and **ignores** `If-Modified-Since`. Its weaknesses are inherent to using wall-clock dates as version identifiers — **one-second granularity**, reliance on fragile filesystem/mtime timestamps, and an inability to express "changed then changed back to identical bytes" — which is exactly why `ETag`/`If-None-Match` was invented. It remains widely used because it's simple, universally supported, and free when a resource already has a meaningful modification time (static files, blog posts, documents).

## What problem does this header solve?

Same core problem as [`If-None-Match`](./If-None-Match.md): when a cached copy's freshness ([`Cache-Control: max-age`](../06-Caching-Headers/Cache-Control.md)) lapses, the cache must decide whether to re-download or cheaply confirm the copy is still current. `If-Modified-Since` provides the cheap confirmation using a **timestamp** rather than an opaque tag: the client says "I have a copy from `Tue, 01 Jul 2026 10:00:00 GMT` — anything newer?" The origin compares that date to the resource's current modification time and answers `304 Not Modified` (no body) if unchanged, or `200` with the new body if it changed.

The reason a *date-based* option exists at all — rather than everyone just using ETags — is that many resources come with a **natural, cheap modification time**: a file's mtime, a database row's `updated_at`, a document's last-edited timestamp. For those, you get validation "for free" without computing a content hash on every request. `If-Modified-Since` lets clients revalidate against that timestamp, saving bandwidth on unchanged resources, and it interoperates with the oldest caches and proxies in existence.

## Why was it introduced?

`If-Modified-Since` predates `ETag`: it was part of **HTTP/1.0 (RFC 1945, 1996)**, the *original* conditional-GET mechanism, carried into HTTP/1.1 and re-specified in **RFC 7232 (2014)** and **RFC 9110 §13.1.3 (2022)**. In the HTTP/1.0 era, the only way to ask "has this changed?" was by date, and `Last-Modified` + `If-Modified-Since` was the whole caching-validation story. HTTP/1.1 introduced `ETag`/`If-None-Match` specifically to fix the date approach's flaws (granularity, clock dependence, semantic-equivalence), and defined that `If-None-Match` **takes precedence** when both are sent — making `If-Modified-Since` the fallback validator. It survives because it's zero-cost for resources with real timestamps and because *every* HTTP implementation understands it, but for anything where correctness under sub-second changes matters, the spec and practice both prefer `ETag`.

## How does it work?

The client sends `If-Modified-Since: <http-date>`. The origin compares that date to the resource's current [`Last-Modified`](../06-Caching-Headers/Last-Modified.md) time:

- **Resource's `Last-Modified` ≤ the `If-Modified-Since` date** (i.e. not changed since) → `304 Not Modified`, **no body**.
- **Resource changed after that date** (or has no reliable time and you decide to send) → `200 OK` with the full body and a fresh `Last-Modified`.

`If-Modified-Since` applies only to `GET`/`HEAD` (safe methods); on other methods it's ignored. It's a **weak** validator by nature — it can't distinguish two different representations that share a second.

```mermaid
sequenceDiagram
    participant B as Browser / cache
    participant O as Origin
    Note over B: Cached copy stale; Last-Modified = Jul 1 10:00:00
    B->>O: GET /article/42<br/>If-Modified-Since: Tue, 01 Jul 2026 10:00:00 GMT
    O->>O: current Last-Modified > that date?
    alt Not changed
        O-->>B: 304 Not Modified (no body)<br/>Last-Modified: Jul 1 10:00:00
        Note over B: Reuse cached bytes, refresh freshness
    else Changed
        O-->>B: 200 OK<br/>Last-Modified: Jul 3 14:22:00 (full body)
        Note over B: Replace cached entry
    end
```

- **Browser behavior:** The browser stores the `Last-Modified` value of a cached response and **automatically** attaches `If-Modified-Since` when revalidating (freshness lapsed, `no-cache`, reload) — *unless* it also has an `ETag`, in which case it sends `If-None-Match` (and may send both). On `304` it reuses the cached body transparently.
- **Server behavior:** The origin emits `Last-Modified`, then on later requests compares the incoming `If-Modified-Since` and returns `304`/`200`. It must have a *reliable, monotonic* modification time or risk serving stale (if mtime jumps backward) or wasteful (if mtime changes without content changing) responses.
- **Proxy behavior:** Shared proxies revalidate upstream by forwarding `If-Modified-Since` and pass `304`/`Last-Modified` through.
- **CDN behavior:** CDNs revalidate stale objects with the origin via `If-Modified-Since` (or `If-None-Match`), collapsing traffic. Sensitive to origin clock skew and mtime consistency across nodes.
- **Reverse proxy behavior:** Nginx answers conditional GETs for static files from `Last-Modified` (file mtime) automatically and forwards `If-Modified-Since` for proxied content.

## HTTP Request Example

Revalidating a cached article by date:

```http
GET /article/42 HTTP/1.1
Host: blog.example.com
If-Modified-Since: Tue, 01 Jul 2026 10:00:00 GMT
```

Both validators present (server will honor `If-None-Match` and ignore the date):

```http
GET /assets/app.css HTTP/1.1
Host: shop.example.com
If-None-Match: "9f2c-a1b2c3"
If-Modified-Since: Tue, 01 Jul 2026 10:00:00 GMT
```

## HTTP Response Example

The first response that seeds the validator:

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Last-Modified: Tue, 01 Jul 2026 10:00:00 GMT
Cache-Control: public, max-age=0, s-maxage=300
Content-Length: 8123
```

A successful revalidation — **no body**:

```http
HTTP/1.1 304 Not Modified
Last-Modified: Tue, 01 Jul 2026 10:00:00 GMT
Cache-Control: public, max-age=0, s-maxage=300
```

Changed — full response with a newer time:

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Last-Modified: Thu, 03 Jul 2026 14:22:00 GMT
Content-Length: 8641
```

## Express.js Example

```js
const express = require('express');
const app = express();

// 1) EXPLICIT date-based revalidation for a DB-backed resource with updated_at.
app.get('/article/:id', async (req, res) => {
  const article = await db.articles.find(req.params.id);
  if (!article) return res.status(404).end();

  // Last-Modified must be an HTTP date (RFC 1123), second-resolution.
  const lastModified = new Date(article.updatedAt);
  res.set('Last-Modified', lastModified.toUTCString());
  res.set('Cache-Control', 'public, max-age=0, s-maxage=300');

  // Compare the client's If-Modified-Since. Note: dates are second-granular, so
  // compare truncated-to-seconds to avoid false 200s from millisecond noise.
  const ims = req.headers['if-modified-since'];
  if (ims) {
    const since = Date.parse(ims);                       // ms since epoch, or NaN
    if (!Number.isNaN(since) && Math.floor(lastModified.getTime() / 1000) <= Math.floor(since / 1000)) {
      return res.status(304).end();                      // unchanged → no body.
    }
  }
  res.type('html').send(renderArticle(article));
});

// 2) Static files: express.static sets Last-Modified from mtime and answers
//    If-Modified-Since automatically. You usually don't hand-code this.
app.use('/assets', express.static('public', {
  maxAge: 0,                 // force revalidation; static() handles If-Modified-Since/If-None-Match.
  lastModified: true,        // default: emit Last-Modified from file mtime.
  etag: true,                // also emit ETag (which will take precedence on revalidation).
}));

app.listen(3000);
```

Why each piece matters: `lastModified.toUTCString()` produces the RFC 1123 date format HTTP requires — sending a non-conforming date makes clients ignore it. The **truncate-to-seconds** comparison is a subtle but real correctness point: `If-Modified-Since` dates have one-second resolution, so if your `updated_at` has milliseconds, a naive `>` comparison can conclude "changed" when only sub-second noise differs, defeating the `304`. Returning `.status(304).end()` (no body) is mandatory. In route 2, `express.static` does all of this for you *and* adds an `ETag`, which the browser will prefer — so date-based revalidation is really the *fallback* path there.

## Node.js Example

Raw `http` — explicit date comparison:

```js
const http = require('http');

http.createServer((req, res) => {
  if (req.url === '/report' && req.method === 'GET') {
    const lastModified = getReportMtime();                 // a Date, e.g. from fs.statSync().mtime
    res.setHeader('Last-Modified', lastModified.toUTCString());
    res.setHeader('Cache-Control', 'public, max-age=0, s-maxage=60');
    res.setHeader('Content-Type', 'text/plain');

    const ims = req.headers['if-modified-since'];
    if (ims) {
      const since = Date.parse(ims);
      // Compare at second resolution (HTTP-date granularity).
      if (!Number.isNaN(since) &&
          Math.floor(lastModified.getTime() / 1000) <= Math.floor(since / 1000)) {
        res.statusCode = 304;
        return res.end();                                  // no body on 304
      }
    }
    res.statusCode = 200;
    return res.end(renderReport());
  }
  res.statusCode = 404;
  res.end();
}).listen(3000);
```

The lesson mirrors Express: parse the date defensively (`NaN` guard), compare at second resolution, and never send a body with `304`.

## React Example

React never sets `If-Modified-Since` — the browser attaches it automatically during revalidation, and `304` is resolved beneath your code:

1. **Automatic date revalidation.** When React `fetch`es a resource that returned `Last-Modified` (but no `ETag`), the browser stores the date and, on revalidating a stale copy, sends `If-Modified-Since` for you; your `.then()` gets the cached body transparently on a `304`.

```jsx
function useArticle(id) {
  const [article, setArticle] = React.useState(null);
  React.useEffect(() => {
    // If the server sent Last-Modified, the browser revalidates with
    // If-Modified-Since automatically when this cached copy goes stale.
    fetch(`/article/${id}`, { cache: 'no-cache' })
      .then(r => r.text())
      .then(setArticle);
  }, [id]);
  return article;
}
```

2. **Why you'd rarely set it manually:** `fetch` gives you the parsed body, not `304` handling, unless you manage a custom cache (e.g. in a service worker) with `cache: 'no-store'` and read `Response.headers.get('last-modified')` yourself.

3. **Prefer ETags when you control the server:** for React data APIs, an `ETag`/`If-None-Match` flow is more robust than dates (no clock/granularity issues), so treat `If-Modified-Since` as the compatibility fallback for timestamped static content.

## Browser Lifecycle

1. **First response** carries `Last-Modified` → the browser stores that date with the cached body.
2. **Fresh copy** → served from cache with no network; `If-Modified-Since` not sent.
3. **Stale / `no-cache` / reload** → the browser revalidates. If the copy has an [`ETag`](../06-Caching-Headers/ETag.md), it sends [`If-None-Match`](./If-None-Match.md) (and may also send `If-Modified-Since`); if only `Last-Modified`, it sends `If-Modified-Since` alone.
4. **`304`** → reuse the stored body, refresh freshness from the response headers.
5. **`200`** → replace the cached entry with the new body + new `Last-Modified`.
6. When both validators are present, the server should honor `If-None-Match` and ignore the date — so the ETag path wins.

## Production Use Cases

- **Static file serving:** the default, zero-cost validator for files (mtime → `Last-Modified`), handled automatically by web servers and `express.static`.
- **Timestamped content APIs:** blog posts, docs, CMS entries with an `updated_at` — revalidate by date without hashing bodies.
- **Legacy/interop clients:** older HTTP clients and proxies that don't handle ETags well still support `If-Modified-Since`.
- **RSS/feed polling:** feed readers poll with `If-Modified-Since` to avoid re-downloading unchanged feeds.
- **CDN origin revalidation** for content whose natural key is a modification time.

## Common Mistakes

- **Sub-second granularity bugs.** A resource that changes twice within the same second looks unchanged → clients serve stale content. Use `ETag` where sub-second correctness matters.
- **mtime resets on deploy/restore.** `rsync`, container rebuilds, and restores bump file mtimes without content changing → spurious `200`s (cache busts) — or, worse, mtime moving *backward* → stale served as fresh. Prefer content hashes for deploy-sensitive assets.
- **Non-RFC-1123 date format.** Emitting a non-standard `Last-Modified` (or comparing a mis-parsed `If-Modified-Since`) breaks revalidation; always format/parse HTTP dates correctly.
- **Comparing at millisecond resolution.** Causes false "changed" results; truncate to seconds.
- **Sending a body with `304`.** Protocol violation.
- **Expecting it to win over `If-None-Match`.** It doesn't — when both are present, the ETag decides.
- **Using it on non-GET/HEAD methods** — it's ignored there; for write preconditions use [`If-Unmodified-Since`](./If-Unmodified-Since.md) / [`If-Match`](./Conditional-Requests-Overview.md).

## Security Considerations

- **Clock skew and time-based logic.** Because it hinges on wall-clock time, server clock skew or a client sending a future date can cause incorrect `304`s (serving stale) — never rely on `If-Modified-Since` for anything security-sensitive (e.g. serving revoked content). Use explicit invalidation/purge for security fixes.
- **Less of a tracking vector than ETags.** `Last-Modified` is content-derived (a timestamp), so it's a weaker supercookie vehicle than a per-user ETag — but still avoid encoding per-user data in it.
- **Not authorization.** A `304`/`200` decision must come *after* access-control checks; the conditional says nothing about who may see the resource.
- **Information disclosure.** A precise `Last-Modified` can leak when content (or a file) last changed, occasionally sensitive; usually benign but worth noting for internal resources.

## Performance Considerations

- **Free validation for timestamped resources.** No per-request hashing (unlike a content-hash `ETag`), so it's cheap for static files and DB rows with `updated_at`.
- **Saves the payload, not the RTT.** Like all revalidation, a `304` still costs one round-trip; for immutable assets, long `max-age` + `immutable` (no revalidation) is better.
- **Granularity limits hit ratio correctness.** Not a throughput issue but a correctness one — sub-second changes and mtime churn reduce the value of caching.
- **Pairs well with `ETag`** where both are cheap: send both; clients use the ETag (robust) and fall back to the date for old caches.

## Reverse Proxy Considerations

Nginx handles `If-Modified-Since` for static files from mtime and can control strictness:

```nginx
server {
  location /assets/ {
    root /var/www;
    # Nginx sets Last-Modified from file mtime and answers If-Modified-Since with 304.
    add_header Cache-Control "public, max-age=0, s-maxage=300";

    # if_modified_since controls how strict the comparison is:
    #   exact  - 304 only if dates match exactly
    #   before - 304 if the file mtime is <= If-Modified-Since (default, lenient)
    #   off    - ignore If-Modified-Since entirely
    if_modified_since before;
  }

  location /api/ {
    proxy_pass http://app_upstream;
    proxy_cache app_cache;
    proxy_cache_valid 200 60s;
    proxy_cache_revalidate on;   # revalidate stale entries upstream (If-Modified-Since/If-None-Match).
  }
}
```

Key points: `if_modified_since` tunes the comparison semantics; `before` (default) is the normal "not changed since" behavior. For a **multi-server** static fleet, mtimes can differ across nodes (deploy artifacts) — a request revalidating against node A's mtime may bust on node B — so serve statics from stable storage or use content-hashed URLs. `proxy_cache_revalidate on` lets nginx refresh stale cache entries with conditional requests.

## CDN Considerations

- **Origin revalidation:** CDNs send `If-Modified-Since` (or `If-None-Match`) to refresh stale objects; a stable, monotonic origin `Last-Modified` keeps this efficient.
- **Clock/mtime consistency:** origins behind a CDN must have synchronized clocks and consistent modification times across nodes, or revalidation churns.
- **ETag preferred at scale:** most CDN setups prefer `ETag` for robustness; `Last-Modified`/`If-Modified-Since` is the fallback for timestamped content.
- **Some CDNs synthesize `Last-Modified`** or normalize it; check vendor behavior if you rely on exact dates.

## Cloud Deployment Considerations

- **Object storage (S3/GCS):** set `Last-Modified` from object upload time and honor `If-Modified-Since` on GET, returning `304`. Re-uploading identical content updates the time and busts date-based caches — use content-hashed keys for immutability.
- **API Gateways:** ensure `If-Modified-Since` is forwarded to the backend and `Last-Modified`/`304` passed back.
- **Serverless:** derive `Last-Modified` from a stored `updated_at` (never ephemeral local mtime), and implement the second-resolution comparison in the handler.
- **Load balancers:** pass conditional headers through untouched; clock skew across instances can affect date logic, so keep NTP tight.

## Debugging

- **Chrome DevTools → Network:** a `304` with tiny Size confirms date revalidation worked; check request `If-Modified-Since` and response `Last-Modified`/status in the Headers tab. "Disable cache" forces full `200`s.
- **curl:** read the time — `curl -sD - -o /dev/null https://host/article/42` — then revalidate — `curl -sD - -o /dev/null -H 'If-Modified-Since: Tue, 01 Jul 2026 10:00:00 GMT' https://host/article/42` (expect `304`). curl also has `-z <file-or-date>` to send `If-Modified-Since` automatically: `curl -z "Tue, 01 Jul 2026 10:00:00" ...`.
- **Postman / Bruno:** capture `Last-Modified` into a variable, echo it in `If-Modified-Since`, assert `res.status === 304`.
- **Node.js:** log `req.headers['if-modified-since']` and the resource's modification time to see comparisons.
- **Express logging:** `app.use((req,res,next)=>{res.on('finish',()=>console.log(req.url,res.statusCode,'ims=',req.headers['if-modified-since'],'lm=',res.getHeader('last-modified')));next();});`
- **Granularity check:** change a resource twice within a second and verify (with an ETag) that the second change is detected — exposing the date validator's blind spot.

## Best Practices

- [ ] Emit a correctly-formatted RFC 1123 [`Last-Modified`](../06-Caching-Headers/Last-Modified.md) and compare `If-Modified-Since` at **second** resolution.
- [ ] Prefer [`ETag`](../06-Caching-Headers/ETag.md)/[`If-None-Match`](./If-None-Match.md) when sub-second correctness or deploy-stability matters; use dates as the cheap fallback.
- [ ] Send **both** validators when both are cheap — clients pick the ETag and fall back to the date.
- [ ] Answer a satisfied `If-Modified-Since` with `304` and **no body**.
- [ ] Ensure modification times are **monotonic and consistent** across a fleet (watch mtime resets on deploy).
- [ ] Give [`If-None-Match`](./If-None-Match.md) precedence when both are present.
- [ ] Don't rely on date validation for security-sensitive freshness — use explicit purge/invalidation.
- [ ] Keep server clocks synchronized (NTP) to avoid skew-induced mis-revalidation.

## Related Headers

- [Last-Modified](../06-Caching-Headers/Last-Modified.md) — the response header this echoes; the resource's modification time.
- [If-None-Match](./If-None-Match.md) — the ETag-based revalidation header that **takes precedence** over `If-Modified-Since`.
- [ETag](../06-Caching-Headers/ETag.md) — the opaque validator that fixes the date approach's weaknesses.
- [If-Unmodified-Since](./If-Unmodified-Since.md) — the write-side date precondition ("act only if *not* changed since"), the mirror of this header.
- [If-Range](./If-Range.md) — can use a date to decide whether a resumed range fetch reuses the cached copy.
- [Cache-Control](../06-Caching-Headers/Cache-Control.md) — decides *whether* to revalidate; `If-Modified-Since` decides *what happens* when you do.
- [Conditional Requests Overview](./Conditional-Requests-Overview.md) — the full model.

## Decision Tree

```mermaid
flowchart TD
    A[Revalidating a cached GET] --> B{Does the copy have an ETag?}
    B -- Yes --> C[Send If-None-Match<br/>(browser may also send If-Modified-Since)]
    C --> D[Server honors If-None-Match, ignores the date]
    B -- No, only Last-Modified --> E[Send If-Modified-Since with the stored date]
    E --> F{Changed since that date?}
    F -- No --> G[304 Not Modified, no body → reuse cache]
    F -- Yes --> H[200 OK, new body + new Last-Modified]
    A --> I{Sub-second changes or deploy mtime churn?}
    I -- Yes --> J[Prefer ETag; dates are unreliable here]
```

## Mental Model

Think of `If-Modified-Since` as asking a shopkeeper, **"Has this display changed since I last saw it on Tuesday morning?"** — you're identifying your cached copy by *when* you got it, not by a version label. If nothing's changed since Tuesday, the shopkeeper just says "nope, same as you saw" (`304`, nothing handed over) and you keep using your copy. If it's been updated, they hand you the new one with a new "last changed" date (`200`). The catch is that "when" is a blunt instrument: if the display was swapped twice on Tuesday *within the same minute*, "since Tuesday morning" can't tell — you might walk away thinking it's unchanged when it isn't (one-second granularity). And if the shop re-dusts the display (a deploy touching file mtimes) without actually changing what's on it, they'll wrongly tell you "yes, it changed" and hand you an identical copy. That imprecision is exactly why the shop eventually switched to putting a precise *edition number* on everything ([`ETag`](../06-Caching-Headers/ETag.md)) — and why, when both a date and an edition number are available, everyone trusts the edition number.
