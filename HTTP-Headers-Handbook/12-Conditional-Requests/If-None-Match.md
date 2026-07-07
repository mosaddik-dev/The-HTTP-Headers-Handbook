# If-None-Match

## Quick Summary

`If-None-Match` is a **request** header carrying one or more [`ETag`](../06-Caching-Headers/ETag.md) validator values (or `*`), used to make a request **conditional**: "only give me a full response if the current version is **none of** these tags." It is the read-side workhorse of HTTP caching — when a browser or cache holds a copy whose freshness has lapsed, it echoes the stored `ETag` back in `If-None-Match`, and the origin replies with a cheap **`304 Not Modified`** (no body) if the tag still matches, or a full **`200`** with new bytes if it doesn't. It is the `ETag`-based counterpart to [`If-Modified-Since`](./If-Modified-Since.md) (which is date-based), and when both are present `If-None-Match` **takes precedence**. It has a second, write-side job with `*`: `If-None-Match: *` means "only succeed if the resource does **not** exist," turning a `PUT`/`POST` into a safe **create-if-absent** that prevents accidental overwrites. Get it right and revalidation costs a header round-trip instead of a full download; misuse it and you either re-download unchanged data forever or clobber resources you meant to create.

## What problem does this header solve?

Freshness lifetimes ([`Cache-Control: max-age`](../06-Caching-Headers/Cache-Control.md)) eventually expire. When they do, a cache must decide: re-download the whole resource (wasteful if it hasn't changed) or ask cheaply whether it changed. `If-None-Match` provides the cheap path. The client sends the validator it already holds; the origin compares it to the current one and, on a match, answers `304 Not Modified` with **no body** — confirming the cached bytes are still correct while transferring only headers. For a large JSON payload or asset that rarely changes, this turns "download 400 KB every time" into "send back a few hundred bytes of headers most of the time."

The second problem is **safe resource creation**. In a `PUT`-to-create API, two clients might both try to create `/users/alice`; without a guard, the second silently overwrites the first. `If-None-Match: *` makes the request succeed **only if the target doesn't already exist**, returning `412 Precondition Failed` otherwise — so "create" never becomes an accidental "replace." This is the mirror image of [`If-Match`](./Conditional-Requests-Overview.md)'s "update only if unchanged."

## Why was it introduced?

`If-None-Match` arrived with HTTP/1.1 (RFC 2068, 1997; RFC 2616, 1999) alongside [`ETag`](../06-Caching-Headers/ETag.md), to fix the weaknesses of the HTTP/1.0 date-based validation ([`If-Modified-Since`](./If-Modified-Since.md) + [`Last-Modified`](../06-Caching-Headers/Last-Modified.md)). Dates have 1-second granularity, depend on fragile filesystem timestamps, and can't express "byte-different but semantically identical." Opaque `ETag` validators sidestep clocks entirely, and `If-None-Match` is how a client presents that validator for comparison. The precondition rules were re-specified cleanly in **RFC 7232 (2014, "Conditional Requests")** and carried into **RFC 9110 §13 (2022, "HTTP Semantics")**. The `*` form and the precedence-over-`If-Modified-Since` rule are both defined there, as is the crucial distinction that for **caching revalidation** the comparison uses the **weak** comparison function (weak and strong tags can match), whereas write preconditions demand strong comparison.

## How does it work?

The client sends `If-None-Match: <etag(s)>`. The origin computes/looks up the resource's current `ETag` and compares:

- **For GET/HEAD (revalidation):** if the current tag matches **any** listed tag (weak comparison) → `304 Not Modified`, no body. If none match → `200 OK` with the full body and a fresh `ETag`.
- **For unsafe methods (PUT/POST/PATCH/DELETE):** if the precondition *fails* (i.e. the resource **does** match one of the tags, or with `*` the resource exists) → `412 Precondition Failed`, and the method is **not** applied.

```mermaid
sequenceDiagram
    participant B as Browser / cache
    participant O as Origin
    Note over B: Cached copy stale; holds ETag "v7"
    B->>O: GET /api/products/42<br/>If-None-Match: "v7"
    O->>O: current ETag == "v7"?
    alt Matches (unchanged)
        O-->>B: 304 Not Modified (no body)<br/>ETag: "v7"
        Note over B: Reuse cached bytes, refresh freshness
    else Changed
        O-->>B: 200 OK<br/>ETag: "v9" (full body)
        Note over B: Replace cached entry
    end
```

- **Browser behavior:** The browser stores the `ETag` with a cached response and **automatically** attaches `If-None-Match` when it revalidates (freshness lapsed, `Cache-Control: no-cache`, or a reload). On `304` it transparently serves the stored body; `fetch` sees a normal successful response. Pure equality echo — the browser never interprets the tag.
- **Server behavior:** The origin generates the `ETag` and evaluates incoming `If-None-Match`, returning `304`/`200` (reads) or `412`/success (writes). It must produce the **same tag for the same representation every time**, across restarts and every node in a fleet, or revalidation always misses.
- **Proxy behavior:** A shared forward proxy revalidates upstream by forwarding `If-None-Match`, and must pass `ETag`/`304` through untouched (opaque).
- **CDN behavior:** CDNs collapse many client conditionals into few origin revalidations (origin shielding), keyed per [`Vary`](../06-Caching-Headers/Vary.md) variant; they're highly sensitive to cross-node ETag inconsistency.
- **Reverse proxy behavior:** Nginx answers/forwards conditional GETs using stored ETags and generates its own for static files.

## HTTP Request Example

Revalidating a cached asset:

```http
GET /assets/app.css HTTP/1.1
Host: shop.example.com
If-None-Match: "9f2c-a1b2c3"
Accept-Encoding: gzip, br
```

A create-if-absent write guard:

```http
PUT /api/users/alice HTTP/1.1
Host: api.example.com
If-None-Match: *
Content-Type: application/json

{"name":"Alice"}
```

Multiple candidate tags (client holds several possible versions):

```http
GET /feed HTTP/1.1
Host: news.example.com
If-None-Match: "v7", "v8", W/"v8-weak"
```

## HTTP Response Example

Successful revalidation — **no body**, same tag echoed:

```http
HTTP/1.1 304 Not Modified
ETag: "9f2c-a1b2c3"
Cache-Control: public, max-age=0, s-maxage=300
```

Content changed — full response, new tag:

```http
HTTP/1.1 200 OK
Content-Type: text/css; charset=utf-8
ETag: "7d1e-ff0099"
Cache-Control: public, max-age=0, s-maxage=300
Content-Length: 41022
```

Create-if-absent failed because the resource already exists:

```http
HTTP/1.1 412 Precondition Failed
Content-Type: application/json

{"error":"already_exists"}
```

## Express.js Example

```js
const express = require('express');
const crypto = require('crypto');
const app = express();
app.use(express.json());

// 1) READ revalidation. Express auto-handles If-None-Match for bodies it generates
//    when an ETag is enabled — but for full control (and fleet-safe tags) do it explicitly.
app.get('/api/products/:id', async (req, res) => {
  const product = await db.products.find(req.params.id);
  if (!product) return res.status(404).end();

  const body = JSON.stringify(product);
  // Fleet-safe strong validator: same bytes → same tag on every node, forever.
  const etag = '"' + crypto.createHash('sha1').update(body).digest('base64') + '"';

  res.set('Cache-Control', 'public, max-age=0, s-maxage=60');
  res.set('ETag', etag);

  // If-None-Match may be a comma list or "*"; parse it, don't ===.
  const inm = req.headers['if-none-match'];
  if (inm) {
    const tags = inm.split(',').map(s => s.trim());
    if (tags.includes('*') || tags.includes(etag)) {
      return res.status(304).end();   // 304 MUST have no body.
    }
  }
  res.type('application/json').send(body);
});

// 2) CREATE-IF-ABSENT with If-None-Match: * — prevents overwriting an existing resource.
app.put('/api/users/:name', async (req, res) => {
  const exists = await db.users.exists(req.params.name);
  const guard = req.headers['if-none-match'];

  if (guard === '*' && exists) {
    // Client asked to create only if absent, but it exists → refuse.
    return res.status(412).json({ error: 'already_exists' });
  }
  const user = await db.users.upsert(req.params.name, req.body);
  res.status(exists ? 200 : 201)
     .set('ETag', '"' + user.version + '"')
     .json(user);
});

// 3) Let Express auto-generate weak ETags + auto-answer If-None-Match for simple routes:
app.set('etag', 'strong'); // upgrade default 'weak' to byte-exact strong tags.

app.listen(3000);
```

Why each piece matters: parsing `If-None-Match` as a list (route 1) is essential — browsers and proxies can send `"a", "b"` or `*`, and a naive `inm === etag` breaks legitimate revalidations, forcing full re-downloads. Returning `304` with `.status(304).end()` (never `res.send()`) is required because **a `304` must carry no body** — a framework that sends one violates the protocol and confuses caches. In route 2, `If-None-Match: *` is the create guard: without it, a `PUT` to an existing name silently overwrites, reintroducing the lost-resource bug. `app.set('etag', 'strong')` (route 3) makes Express both emit strong tags and *automatically* answer matching `If-None-Match` with `304` for bodies it generates — convenient, but it uses a body hash, not necessarily your domain version, so for APIs you often want the explicit approach in route 1.

## Node.js Example

Raw `http` gives you nothing automatically — you own the comparison:

```js
const http = require('http');
const crypto = require('crypto');

http.createServer((req, res) => {
  if (req.url === '/api/config' && req.method === 'GET') {
    const body = JSON.stringify({ theme: 'dark', flags: ['beta'] });
    // Content hash on the ORIGINAL bytes: deterministic, fleet-safe, encoding-independent.
    const etag = '"' + crypto.createHash('sha1').update(body).digest('hex') + '"';

    res.setHeader('ETag', etag);
    res.setHeader('Cache-Control', 'public, max-age=0, s-maxage=300');
    res.setHeader('Content-Type', 'application/json');

    const inm = req.headers['if-none-match'];
    const tags = inm ? inm.split(',').map(s => s.trim()) : [];
    if (tags.includes('*') || tags.includes(etag)) {
      res.statusCode = 304;   // matched → confirm without payload.
      return res.end();       // no body on 304.
    }
    res.statusCode = 200;
    return res.end(body);
  }
  res.statusCode = 404;
  res.end();
}).listen(3000);
```

The lesson: `If-None-Match` handling is a small, explicit comparison, but the correctness (list parsing, `*`, no-body-on-304, fleet-consistent tags) is exactly what frameworks paper over.

## React Example

React never sets `If-None-Match` directly — the browser attaches it automatically during revalidation, and `304` is resolved *below* your code, so `fetch` transparently returns the cached body. The relationship is indirect but real:

1. **Automatic revalidation.** When React `fetch`es an endpoint that returned an `ETag`, the browser stores it and, on the next fetch of a stale copy (or with `cache: 'no-cache'`), sends `If-None-Match` for you. Your `.then(r => r.json())` gets the cached body on a `304` with no extra work.

```jsx
function useConfig() {
  const [config, setConfig] = React.useState(null);
  React.useEffect(() => {
    // 'no-cache' forces a conditional revalidation (If-None-Match) even when the
    // cached copy is still fresh — pair with a server sending ETag + no-cache.
    fetch('/api/config', { cache: 'no-cache' })
      .then(r => r.json())
      .then(setConfig);
  }, []);
  return config;
}
```

2. **Manual conditional fetch (rare).** If you *do* want to control it (e.g. a custom cache in a service worker), you can read `Response.headers.get('etag')`, store it, and send it back yourself: `fetch(url, { headers: { 'If-None-Match': savedEtag } })` — then handle `res.status === 304` explicitly (note: you must set `cache: 'no-store'` or the browser's own cache may interfere).

3. **Create-if-absent from the client:** for a `PUT` create, send `If-None-Match: '*'` and treat `412` as "already exists" in your mutation error handling.

## Browser Lifecycle

1. **First response** carries `ETag` → the browser stores it with the cached body and freshness metadata.
2. **Fresh copy** → served with no network; `If-None-Match` never sent.
3. **Stale / `no-cache` / reload** → the browser issues a conditional GET, auto-attaching `If-None-Match: <stored tag>` (plus [`If-Modified-Since`](./If-Modified-Since.md) if it also has [`Last-Modified`](../06-Caching-Headers/Last-Modified.md)).
4. **`304`** → reuse stored body, refresh freshness from the `304`'s headers. DevTools shows `304` and a tiny transfer size.
5. **`200`** → replace the cached entry with the new body + new `ETag`.
6. For ranged/resumable transfers, the related [`If-Range`](./If-Range.md) uses the same tag to decide whether a partial fetch can stitch onto the same version.

## Production Use Cases

- **API response revalidation:** `Cache-Control: no-cache` + strong content-hash `ETag` so clients revalidate constantly but pay only a header RTT + `304` when data is unchanged — huge for large JSON.
- **Static assets behind a CDN:** the CDN revalidates stale objects with the origin via `If-None-Match`, collapsing traffic.
- **Create-if-absent APIs:** `If-None-Match: *` on `PUT` to guarantee "create, never overwrite" (idempotent-safe resource creation).
- **Bandwidth-constrained clients** (mobile, IoT): revalidation avoids re-downloading unchanged payloads.
- **GraphQL/REST gateways:** compute an `ETag` from the serialized response so even computed responses can be `304`'d.

## Common Mistakes

- **Naive `===` instead of list parsing.** `If-None-Match` can be `"a", "b"` or `*`; a strict equality check breaks legitimate revalidation and forces full downloads.
- **Sending a body with `304`.** A `304` MUST have no body; using `res.send()` instead of `.status(304).end()` is a protocol violation.
- **Non-deterministic ETags across a fleet.** mtime/inode-based tags differ per node → `If-None-Match` never matches → `304` never happens. Use content hashes or shared version counters.
- **Using `If-None-Match: *` on a plain GET** expecting cache behavior — on GET, `*` means "give me the body only if the resource doesn't exist," which is almost never what you want for reads.
- **Confusing it with [`If-Match`](./Conditional-Requests-Overview.md).** `If-None-Match` = "act if *none* match" (revalidate / create-if-absent); `If-Match` = "act if it *does* match" (update-if-unchanged). Swapping them breaks concurrency logic.
- **Ignoring precedence.** When both `If-None-Match` and `If-Modified-Since` are present, evaluate `If-None-Match` and ignore the date; doing the reverse can serve stale content.
- **Unquoted tags.** ETags are quoted; comparing against an unquoted value never matches.

## Security Considerations

- **ETag tracking (supercookies).** Because the browser faithfully echoes a per-user `ETag` in `If-None-Match` on every revalidation, a server can assign each visitor a unique tag and track them even after cookies are cleared. Derive ETags **only** from resource content, never from user identity. Privacy-hardened setups may strip ETags for this reason.
- **Information disclosure via tags.** `"inode-mtime-size"` tags leak filesystem metadata and sizes; prefer opaque content hashes.
- **Not an authorization mechanism.** A matching `If-None-Match` proves the client saw *a* version, not that it's *authorized*. Always enforce auth independently; `412`/`304` decisions must come *after* access checks.
- **DoS via many/huge tags.** A client can send a long `If-None-Match` list; bound the parse work and reject absurd inputs.
- **Cache-key correctness.** Revalidation for the wrong [`Vary`](../06-Caching-Headers/Vary.md) variant can serve mismatched bytes; ensure the variant is keyed before comparing tags.

## Performance Considerations

- **`304` saves the payload, not the RTT.** For truly immutable assets, `Cache-Control: immutable` + long `max-age` (skip revalidation entirely) beats even a `304`. For mutable data, `304` beats re-downloading.
- **ETag generation cost.** Hashing every response body adds CPU; for hot endpoints, cache the computed tag alongside the data; for large static files, size+mtime is cheaper (but not fleet-safe — trade-off).
- **Fleet consistency drives hit ratio.** Inconsistent tags silently destroy revalidation efficiency across CDN/browser; content hashes are a pure win.
- **Conditional requests reduce origin egress** dramatically for large, rarely-changing resources — the core economic reason CDNs revalidate with `If-None-Match`.

## Reverse Proxy Considerations

Nginx forwards/answers conditional GETs using ETags:

```nginx
server {
  location /assets/ {
    root /var/www;
    etag on;               # nginx generates ETag: "<mtime-hex>-<size-hex>" for static files.
    add_header Cache-Control "public, max-age=0, s-maxage=300";
    # Nginx answers a matching If-None-Match with 304 automatically for static files.
  }

  location /api/ {
    proxy_pass http://app_upstream;
    # Forward the client's If-None-Match upstream and pass ETag/304 back untouched.
    proxy_cache app_cache;
    proxy_cache_valid 200 60s;
    proxy_cache_revalidate on;   # nginx revalidates stale entries upstream with If-None-Match.
    add_header X-Cache-Status $upstream_cache_status;
  }
}
```

Key points: `proxy_cache_revalidate on` makes nginx use `If-None-Match`/`If-Modified-Since` to refresh stale cached entries instead of refetching whole bodies. For a **multi-server** static fleet, mtime-based tags differ per node — serve statics from a CDN/object store with stable tags or use content-hashed filenames. Never let a body-transforming proxy keep a strong tag it no longer matches.

## CDN Considerations

- **Origin shielding:** the CDN sends `If-None-Match` to the origin for stale objects and returns `304` to collapse client traffic; a stable origin `ETag` keeps hit ratio high.
- **Cross-node inconsistency is the #1 bug:** autoscaled origins emitting per-instance tags make every CDN revalidation miss and re-pull the full body — use content hashes.
- **Some CDNs synthesize ETags** when the origin omits one, enabling `304`s even for `Last-Modified`-only origins.
- **Variant correctness:** the CDN must match the right [`Vary`](../06-Caching-Headers/Vary.md) variant before comparing tags, or a compressed/uncompressed mixup breaks validation.

## Cloud Deployment Considerations

- **Object storage (S3/GCS/Azure Blob):** set `ETag` automatically and honor `If-None-Match` on GET (S3 supports it), returning `304`. Note S3 multipart ETags aren't a whole-object MD5 — don't assume tag == content hash.
- **API Gateways:** ensure they forward `If-None-Match` to the backend and don't strip `ETag`; some need explicit config to pass conditional headers.
- **Serverless:** compute the tag from response content or a stored version (never from ephemeral local mtime), and implement the `If-None-Match` comparison in the handler.
- **Load balancers:** pass conditional headers through untouched.

## Debugging

- **Chrome DevTools → Network:** a `304` row with a tiny Size confirms revalidation. Click a request → Headers to see request `If-None-Match` and response `ETag`/status. "Disable cache" forces full `200`s to inspect fresh tags.
- **curl:** read the tag first — `curl -sD - -o /dev/null https://host/app.css` — then revalidate — `curl -sD - -o /dev/null -H 'If-None-Match: "9f2c-a1b2c3"' https://host/app.css` (expect `304`). Test create-guard with `-X PUT -H 'If-None-Match: *'` (expect `412` if it exists).
- **Postman / Bruno:** capture the `ETag` into a variable and echo it in `If-None-Match` on the next request; assert `res.status === 304`.
- **Node.js:** log `req.headers['if-none-match']` and the tag you compute, on every request, to see matches/misses.
- **Express logging:** `app.use((req,res,next)=>{res.on('finish',()=>console.log(req.url,res.statusCode,'inm=',req.headers['if-none-match'],'etag=',res.getHeader('etag')));next();});`
- **Fleet check:** curl the same asset across nodes and confirm identical `ETag`s (mismatched tags = broken revalidation).

## Best Practices

- [ ] Parse `If-None-Match` as a **comma list** and handle `*`; never use bare `===`.
- [ ] Answer a matching `If-None-Match` with `304` and **no body**.
- [ ] Generate [`ETag`](../06-Caching-Headers/ETag.md)s **deterministically from content** (or a shared version), fleet-consistently.
- [ ] Pair with an explicit [`Cache-Control`](../06-Caching-Headers/Cache-Control.md) freshness directive (validators without freshness force revalidation every time).
- [ ] Use `If-None-Match: *` for **create-if-absent** writes → `412` if the resource exists.
- [ ] Give `If-None-Match` **precedence** over [`If-Modified-Since`](./If-Modified-Since.md) when both are present.
- [ ] Keep ETags opaque and content-derived — never encode user identity (tracking) or filesystem metadata (disclosure).
- [ ] Enforce authorization **before** evaluating the precondition.
- [ ] For immutable content-hashed URLs, skip revalidation entirely (`immutable` + long `max-age`).

## Related Headers

- [ETag](../06-Caching-Headers/ETag.md) — the validator this header carries; the origin generates it and `If-None-Match` echoes it.
- [If-Match](./Conditional-Requests-Overview.md) — the write-side opposite ("act if it *does* match") for optimistic concurrency; mismatch → `412`.
- [If-Modified-Since](./If-Modified-Since.md) — the date-based revalidation header; `If-None-Match` takes precedence when both are sent.
- [Last-Modified](../06-Caching-Headers/Last-Modified.md) — the date validator paired with `If-Modified-Since`.
- [Cache-Control](../06-Caching-Headers/Cache-Control.md) — decides *whether* to revalidate; `If-None-Match` decides *what happens* when you do.
- [If-Range](./If-Range.md) — uses an ETag to decide whether a resumed range fetch can reuse the cached copy.
- [Vary](../06-Caching-Headers/Vary.md) — selects the right variant before tag comparison.
- [Conditional Requests Overview](./Conditional-Requests-Overview.md) — the full `304`/`412` model.

## Decision Tree

```mermaid
flowchart TD
    A[Sending If-None-Match] --> B{GET/HEAD or write?}
    B -- GET/HEAD --> C[Echo the cached ETag(s)<br/>= revalidate]
    C --> D{Server: current tag ∈ list?}
    D -- Yes --> E[304 Not Modified, no body → reuse cache]
    D -- No --> F[200 OK, new body + new ETag]
    B -- PUT/POST create --> G[Send If-None-Match: *]
    G --> H{Resource already exists?}
    H -- Yes --> I[412 Precondition Failed → do NOT overwrite]
    H -- No --> J[Create it → 201]
```

## Mental Model

Think of `If-None-Match` as showing the librarian the **edition number of the book you already have** and asking: "Do you have anything *other than* edition `v7`? If not, don't bother handing me a copy — I'll keep mine." If the current edition is still `v7`, the librarian just nods — "yep, yours is current" — and hands you nothing (`304`, no book changes hands, but you've *confirmed* it's up to date). If a new edition exists, they hand you the fresh copy with its new number (`200` + new `ETag`). The `*` form flips the question into the acquisitions desk: "Only shelve my new book if you *don't already have one* by this title" — if a copy's already there, they refuse (`412`) so you never accidentally overwrite the existing one. The edition number is deliberately meaningless to you — you only ever compare it, never read into it — which is exactly why the library is free to number editions however it likes, as long as the *same book* always carries the *same number* on every branch.
