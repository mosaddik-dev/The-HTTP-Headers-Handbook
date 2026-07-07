# Surrogate-Control / Surrogate-Key

## Quick Summary

`Surrogate-Control` and `Surrogate-Key` are **response** headers aimed specifically at **"surrogates" — i.e. CDNs and reverse-proxy caches** — as opposed to browsers. They solve a fundamental tension in caching: you often want a resource cached **aggressively at the CDN edge** but **not (or briefly) in the browser**, and you want to **purge grouped content instantly** without waiting for TTLs. `Surrogate-Control` (from the Edge Architecture Specification, "Edge-Control") lets the origin give the CDN caching directives that are **separate from and invisible to the browser** — e.g. `Surrogate-Control: max-age=86400` tells the CDN to cache for a day while your [`Cache-Control`](./Cache-Control.md) tells the browser something shorter. `Surrogate-Key` (Fastly's term; Cloudflare calls the equivalent **Cache-Tag**, Akamai uses **Edge-Cache-Tag**) attaches one or more **tags** to a response so you can later **purge every object sharing a tag in one API call** — the killer feature for cache invalidation ("purge everything tagged `product-42`" after an edit). Both are **removed by the CDN before the response reaches the browser** (the CDN consumes them). They're vendor-flavored and not a single universal standard, but the *patterns* — edge-only TTLs and tag-based purging — are central to production CDN strategy.

## What problem does this concept solve?

Two hard caching problems that plain [`Cache-Control`](./Cache-Control.md) handles poorly.

**1. Diverging edge vs browser caching.** You frequently want *long* caching at the CDN (to maximize edge hit ratio and shield the origin) but *short or no* caching in the browser (so users get updates quickly, or so you can purge and have users see fresh content immediately). `Cache-Control` has `s-maxage` (shared caches) vs `max-age` (all caches), which helps — but `Surrogate-Control` goes further: it gives the CDN a directive that is **completely separate** from what the browser sees, and it's **stripped before delivery**, so the browser never even knows the edge policy. This cleanly decouples the two tiers.

**2. Instant, grouped invalidation.** TTL-based expiry is a blunt instrument: if you cache a product page for 24 hours and the price changes, you either serve stale data or set short TTLs that hurt hit ratio. What you really want is **event-driven purging**: "when product 42 changes, purge every cached page/asset related to product 42 — now." Doing that by URL is impractical (a product may appear on dozens of pages). `Surrogate-Key`/Cache-Tag solves it by **tagging** responses (`Surrogate-Key: product-42 category-shoes`) so a single purge-by-tag call invalidates the whole group instantly, letting you cache aggressively *and* stay fresh.

## Why was it introduced?

`Surrogate-Control` comes from the **Edge Architecture Specification (ESI / "Edge-Control", ~2001)**, a joint effort by Akamai, Oracle, and others to give origins a way to control edge/surrogate caches independently of downstream (browser) caches — recognizing early that CDN caching and browser caching have different goals. It was designed to be **consumed and removed by the surrogate**, invisible downstream. **Surrogate-key / cache-tag purging** emerged later as CDNs matured and operators demanded **event-driven invalidation** instead of TTL-only expiry — Fastly popularized `Surrogate-Key` with instant purge-by-tag, and other CDNs built equivalents (Cloudflare **Cache-Tag** + tag purge on Enterprise, Akamai **Edge-Cache-Tag**, etc.). There's no single IETF standard covering the tag headers — they're **vendor conventions** implementing the same pattern — so exact header names and semantics differ per CDN, but the two ideas (edge-only control, tag-based purge) are universal in modern CDN practice.

## How does it work?

The origin emits `Surrogate-Control` (edge caching directives) and/or `Surrogate-Key`/Cache-Tag (tags). The CDN **applies** the directives, **records** the tags against the cached object, and **strips** these headers so the browser only sees [`Cache-Control`](./Cache-Control.md). Later, a purge-by-tag API call evicts all objects with that tag.

```mermaid
sequenceDiagram
    participant O as Origin
    participant CDN as CDN (surrogate)
    participant B as Browser
    O-->>CDN: 200 OK<br/>Cache-Control: max-age=60 (browser)<br/>Surrogate-Control: max-age=86400 (edge)<br/>Surrogate-Key: product-42 category-shoes
    Note over CDN: Cache 24h at edge; tag object with keys.<br/>STRIP Surrogate-* headers.
    CDN-->>B: 200 OK<br/>Cache-Control: max-age=60 (only this reaches browser)
    Note over O,CDN: Later: product 42 changes
    O->>CDN: PURGE by key "product-42" (API call)
    Note over CDN: Instantly evict ALL objects tagged product-42
```

- **Browser behavior:** Never sees `Surrogate-*` (stripped by the CDN). It obeys the [`Cache-Control`](./Cache-Control.md) the CDN forwards. So the browser's cache policy is independent of the edge's.
- **Server behavior:** The origin sets `Surrogate-Control` for edge TTL/behavior, `Surrogate-Key`/Cache-Tag for grouping, and a *separate* `Cache-Control` for the browser. It triggers purges via the CDN's API on content changes.
- **CDN behavior (the "surrogate"):** Applies `Surrogate-Control` (taking precedence over `Cache-Control` for its own caching), indexes objects by tag, strips the headers, and executes tag purges. **Vendor-specific** (Fastly `Surrogate-Key`, Cloudflare `Cache-Tag`, Akamai `Edge-Cache-Tag`).
- **Reverse proxy behavior:** Some caching proxies (Varnish, Squid) understand `Surrogate-Control`/ESI; others need explicit config. Varnish uses its own VCL + tag mechanisms.

## HTTP Request Example

These are **response** headers; there's no request form. A purge is a separate **API call** to the CDN, e.g. (Fastly):

```http
POST /service/SERVICE_ID/purge/product-42 HTTP/1.1
Host: api.fastly.com
Fastly-Key: <api-token>
```

## HTTP Response Example

Origin response with divergent edge/browser caching + tags (what the CDN receives):

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Cache-Control: public, max-age=60
Surrogate-Control: max-age=86400, stale-while-revalidate=3600
Surrogate-Key: product-42 category-shoes brand-acme
```

What the **browser** actually receives (CDN stripped the `Surrogate-*` headers):

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Cache-Control: public, max-age=60
Age: 42
```

Cloudflare-flavored equivalent (Cache-Tag):

```http
HTTP/1.1 200 OK
Cache-Control: public, max-age=60
Cache-Tag: product-42,category-shoes,brand-acme
```

## Express.js Example

```js
const express = require('express');
const app = express();

// 1) Diverging edge vs browser caching + tagging for purge.
app.get('/products/:id', async (req, res) => {
  const product = await db.products.find(req.params.id);

  // Browser: short cache so users get updates quickly.
  res.set('Cache-Control', 'public, max-age=60');
  // CDN edge: cache HARD (24h) to shield the origin; separate from the browser.
  res.set('Surrogate-Control', 'max-age=86400, stale-while-revalidate=3600');
  // Tag the response so we can purge by product/category/brand later.
  res.set('Surrogate-Key', `product-${product.id} category-${product.categoryId} brand-${product.brandId}`);
  // (Cloudflare equivalent: res.set('Cache-Tag', `product-${product.id},...`))

  res.type('html').send(renderProduct(product));
});

// 2) On change, PURGE by tag via the CDN API — instant, grouped invalidation.
app.post('/admin/products/:id', async (req, res) => {
  await db.products.update(req.params.id, req.body);
  // Purge every cached object tagged product-<id> across the edge, immediately.
  await fastlyPurgeByKey(`product-${req.params.id}`); // your CDN API wrapper
  res.json({ ok: true });
});

// 3) A helper illustrating the CDN purge API call (Fastly example).
async function fastlyPurgeByKey(key) {
  await fetch(`https://api.fastly.com/service/${SERVICE_ID}/purge/${key}`, {
    method: 'POST',
    headers: { 'Fastly-Key': process.env.FASTLY_TOKEN },
  });
}

app.listen(3000);
```

Why each piece matters: the two separate directives in step 1 are the whole point — `Cache-Control: max-age=60` governs the **browser** (quick updates), while `Surrogate-Control: max-age=86400` tells the **CDN** to cache for a full day (high edge hit ratio, origin shielding), and the CDN *strips* the Surrogate header so these policies never conflict. The `Surrogate-Key` tags are what make step 2's **instant, grouped purge** possible: when a product changes, one `purge/product-42` call evicts *every* cached page/fragment tagged with it — no need to enumerate URLs, no waiting for the 24h TTL. This is the combination that lets you **cache aggressively AND stay fresh**, which plain TTL-based `Cache-Control` can't achieve.

## Node.js Example

Raw `http`:

```js
const http = require('http');

http.createServer((req, res) => {
  // Edge: long; browser: short. CDN strips Surrogate-* before the browser sees it.
  res.setHeader('Cache-Control', 'public, max-age=60');
  res.setHeader('Surrogate-Control', 'max-age=86400');
  res.setHeader('Surrogate-Key', 'product-42 homepage');
  res.setHeader('Content-Type', 'text/html; charset=utf-8');
  res.end('<!doctype html><h1>Product 42</h1>');
}).listen(3000);

// Purge (pseudo): POST to your CDN's purge-by-key endpoint when data changes.
```

The essentials: set browser `Cache-Control` + edge `Surrogate-Control` + `Surrogate-Key` tags, then purge by key on change.

## React Example

React never sets these headers (server/CDN concern), but the *pattern* directly shapes React app freshness:

1. **Aggressive edge caching with instant updates.** Your SSR/Next.js pages (or API responses feeding the React app) can be cached hard at the edge via `Surrogate-Control`, giving fast global loads, while a tag-purge on content change makes users see fresh content immediately — better than short TTLs that both hurt performance and still lag.

2. **Tag-based revalidation in frameworks.** Next.js's `revalidateTag()` / on-demand revalidation is conceptually the *same idea* as `Surrogate-Key` purging (tag content, invalidate by tag on change) — and on some hosts it maps onto CDN cache-tag purging under the hood. React devs benefit from tagging strategy even without touching the raw headers.

```jsx
// Conceptual: a Next.js route handler tagging cached data, then revalidating by tag.
// (Framework-level analog of Surrogate-Key + purge-by-tag.)
// import { revalidateTag } from 'next/cache';
// After a product update: revalidateTag(`product-${id}`);
```

3. **Nothing to set client-side.** The browser never sees `Surrogate-*`; your React data-fetching just gets fresh content because the edge was purged.

## Browser Lifecycle

1. The origin sends `Surrogate-Control`/`Surrogate-Key` **to the CDN**.
2. The CDN applies edge TTL/behavior, tags the object, and **strips** the `Surrogate-*` headers.
3. The **browser** receives only [`Cache-Control`](./Cache-Control.md) (and [`Age`](./Age.md), etc.) — it has no knowledge of edge policy or tags.
4. The browser caches per its `Cache-Control`; the edge caches per `Surrogate-Control`.
5. A tag-purge (out of band, via API) evicts edge objects; the next request repopulates the edge, and browsers get fresh content once their own (short) `Cache-Control` lapses.
6. Net effect: the browser lifecycle is entirely governed by `Cache-Control`; `Surrogate-*` is invisible to it.

## Production Use Cases

- **Origin shielding with fast updates:** cache hard at the edge (`Surrogate-Control: max-age` large), keep browser TTL short, purge-by-tag on change.
- **E-commerce:** tag product/category/brand pages; purge all pages showing a product when its price/stock changes.
- **CMS/publishing:** tag articles by author/section/tag; purge on edit/unpublish instantly.
- **Personalized-but-cacheable shells:** cache the shell at the edge with tags, hydrate personalization client-side.
- **Diverging tiers:** long edge cache + `no-cache`/short browser cache for content that must revalidate per user but is shareable at the edge.
- **Bulk invalidation:** purge an entire section (`section-news`) in one call after a site-wide change.

## Common Mistakes

- **Assuming a universal standard.** Header names/semantics are **vendor-specific** (`Surrogate-Key` vs `Cache-Tag` vs `Edge-Cache-Tag`); check your CDN's docs.
- **Expecting the browser to see/act on `Surrogate-*`.** The CDN strips them; they never reach the browser.
- **Forgetting the separate `Cache-Control`.** If you only set `Surrogate-Control`, the browser may cache per defaults; set both intentionally.
- **Over-tagging or under-tagging.** Too few tags → can't purge granularly; too many/huge tag sets → operational overhead and header bloat. Tag by the entities you actually invalidate on.
- **Not purging on change.** Aggressive edge TTLs without event-driven purge = stale content. The purge is the other half of the pattern.
- **Purge API rate/consistency assumptions.** Tag purges are usually fast but not always instantaneous globally; design for near-real-time, not guaranteed-instant.
- **Leaking internal keys.** Tags can reveal internal IDs if not stripped (CDNs strip them, but verify).

## Security Considerations

- **CDNs strip `Surrogate-*`/tags** before delivery, so internal tag values (which may contain IDs) aren't exposed to clients — but **verify** your CDN does this; a misconfiguration could leak internal identifiers.
- **Purge API is powerful — protect it.** Anyone who can call purge-by-tag can evict your cache (a potential DoS/origin-overload vector) or force revalidation storms. Secure purge credentials tightly (least privilege, rotation).
- **Cache poisoning:** as with all caching, ensure correct cache keys/[`Vary`](./Vary.md) so tagged objects don't serve the wrong content; tags don't replace correct keying.
- **Don't tag with sensitive data** even though it's stripped — minimize what could leak if stripping failed.
- **Diverging tiers and private data:** never edge-cache personalized/private content via `Surrogate-Control` unless the cache key fully isolates users; prefer `private`/no edge caching for such content.

## Performance Considerations

- **This is a performance-and-freshness superpower:** long edge caching (high hit ratio, origin offload, low latency) **combined with** instant purge (freshness) — you no longer trade one for the other.
- **Origin shielding:** aggressive `Surrogate-Control` TTLs drastically cut origin load.
- **Purge efficiency:** tag-based purge invalidates many objects with one call, far cheaper/faster than per-URL purges.
- **`stale-while-revalidate` at the edge** (via `Surrogate-Control`) serves instantly while refreshing, smoothing purges/expiries.
- **Header bloat:** very large `Surrogate-Key` sets add response size to the origin→CDN hop (stripped before the browser); keep tag sets reasonable.

## Reverse Proxy Considerations

Varnish (a common surrogate) implements the pattern via VCL and its own tag/ban mechanisms; classic Nginx has limited native support:

```nginx
# Nginx: no native Surrogate-Key purging in open-source; you'd emit the headers for a
# downstream CDN, or use a module/Nginx Plus for tag-like purging.
server {
  location /products/ {
    proxy_pass http://app_upstream;
    # The app sets Surrogate-Control/Surrogate-Key; the CDN in front consumes them.
    # If Nginx itself is the cache, use proxy_cache_purge (Plus) or cache-key strategies.
  }
}
```

Varnish sketch (concept): store the surrogate keys on the object and `ban`/purge by key. Key points: whether your surrogate is a CDN or Varnish, the origin emits the headers and the surrogate consumes/strips them and implements purge. Open-source Nginx lacks native tag-purge — you typically rely on the CDN or Varnish for this.

## CDN Considerations

- **Vendor-specific — know your CDN:**
  - **Fastly:** `Surrogate-Control` (edge caching) + `Surrogate-Key` (tags) + instant purge-by-key API. The canonical implementation.
  - **Cloudflare:** `Cache-Tag` header (Enterprise) + tag purge; `Cache-Control`/edge TTL settings.
  - **Akamai:** `Edge-Control` / `Edge-Cache-Tag` + Fast Purge by tag.
  - **CloudFront:** invalidation by path (less granular tag support historically).
- **All strip the surrogate/tag headers** before responding to the browser.
- **Purge latency:** usually sub-second to seconds globally; design for near-instant, verify SLAs.
- **Tag limits:** CDNs cap the number/size of tags per object; stay within them.
- **Precedence:** `Surrogate-Control` typically overrides `Cache-Control` *at the edge*; confirm per vendor.

## Cloud Deployment Considerations

- **Framework integration:** Next.js on-demand revalidation / `revalidateTag`, and similar, map onto CDN cache-tag purging on supporting hosts — use the framework abstraction where available.
- **Managed platforms (Vercel/Netlify/Cloudflare):** expose tag-based/on-demand invalidation; prefer their documented mechanism over hand-rolling headers.
- **Secure the purge pipeline:** purge credentials in your deploy/CMS/webhook flow must be protected (they can evict your whole cache).
- **Object storage + CDN:** tag responses at the origin/edge worker; purge by tag on content changes.
- **Observability:** monitor edge hit ratio and purge events to tune TTLs and tagging.

## Debugging

- **curl (see what the origin emits):** `curl -sD - -o /dev/null https://origin/products/42 | grep -i 'surrogate\|cache-tag\|cache-control'` — verify the surrogate headers are present origin-side.
- **curl through the CDN:** the same request via the public URL should **not** show `Surrogate-*` (stripped); you'll see `Cache-Control` + `Age` + vendor cache-status.
- **Verify purge:** note an object's `Age`/cache-status, trigger a tag purge, then re-request — `Age` should reset and cache-status show a MISS/refresh.
- **Vendor cache-status headers:** `cf-cache-status` (Cloudflare), `X-Cache`/`X-Served-By` (Fastly), `X-Cache` (CloudFront) to confirm edge behavior.
- **Tag audit:** ensure the intended tags are applied (Fastly can echo surrogate keys in debug mode).
- **Confirm stripping:** double-check the browser response has no `Surrogate-*`/tag headers (no internal-ID leakage).

## Best Practices

- [ ] Use `Surrogate-Control` for **edge** TTL/behavior and a separate [`Cache-Control`](./Cache-Control.md) for the **browser** — decouple the tiers.
- [ ] **Tag** responses (`Surrogate-Key`/`Cache-Tag`) by the entities you invalidate on (product/category/author/section).
- [ ] **Purge by tag on change** — aggressive edge TTL + event-driven purge is the winning combo.
- [ ] Match the **exact header names/semantics** of your CDN (Fastly/Cloudflare/Akamai differ).
- [ ] Keep tag sets **reasonable** (respect CDN limits; avoid bloat).
- [ ] **Secure the purge API** (least-privilege credentials) — it can evict your whole cache.
- [ ] Add `stale-while-revalidate` at the edge to smooth purges/expiries.
- [ ] Verify the CDN **strips** surrogate headers (no internal-ID leakage) and applies the intended precedence.
- [ ] Never edge-cache private/personalized content unless the cache key fully isolates users.

## Related Headers

- [Cache-Control](./Cache-Control.md) — the browser-facing caching directives; `Surrogate-Control` is its edge-only counterpart (with `s-maxage` as the standardized middle ground).
- [Age](./Age.md) — how old a cached (edge) copy is; useful for verifying purges/TTLs.
- [Vary](./Vary.md) — cache-key correctness; tags don't replace correct keying.
- [ETag](./ETag.md) / [Last-Modified](./Last-Modified.md) — validators for revalidation at the edge.
- [Expires](./Expires.md) — legacy freshness; superseded by `Cache-Control`/`Surrogate-Control`.
- [CDN Caching Overview](../15-CDNs/CDN-Caching-Overview.md) — the broader CDN caching model.
- [Cache Keys and Vary](../15-CDNs/Cache-Keys-and-Vary.md) — how edge cache keys interact with tagging.

## Decision Tree

```mermaid
flowchart TD
    A[Want aggressive edge caching?] --> B{Different TTL for edge vs browser?}
    B -- Yes --> C[Surrogate-Control (edge) + Cache-Control (browser)]
    B -- No --> D[Cache-Control with s-maxage may suffice]
    C --> E{Need instant invalidation on change?}
    D --> E
    E -- Yes --> F[Tag with Surrogate-Key/Cache-Tag<br/>+ purge-by-tag on change]
    E -- No --> G[TTL-based expiry is fine]
    F --> H[Match your CDN's exact header + purge API]
```

## Mental Model

Think of `Surrogate-Control`/`Surrogate-Key` as **special instructions written on the shipping crate for the regional warehouse (the CDN) — instructions the warehouse reads, acts on, and then peels off before the goods go to the customer.** Ordinary handling instructions on the *product itself* ([`Cache-Control`](./Cache-Control.md)) are seen by the end customer, but the warehouse also gets its *own* private note: "keep this stocked for a month" (`Surrogate-Control`) — even if the product's consumer-facing tag says "use within a day." That lets the warehouse keep shelves full (fast local delivery, less strain on the distant factory) while customers still get frequent refreshes. The stroke of genius is the **color-coded inventory tags** (`Surrogate-Key`): every crate gets stickers like "product-42," "brand-acme," "aisle-shoes." When the factory recalls product 42, it doesn't hunt down each crate by serial number — it radios the warehouse "pull everything tagged *product-42*," and in one sweep every affected crate is yanked from every shelf instantly. That combination — *stock aggressively, but recall in one command* — is what lets a CDN be both blazing fast and always fresh, escaping the old dilemma where you had to choose one or the other.
