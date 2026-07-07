# X-Forwarded-Host

## Quick Summary

`X-Forwarded-Host` (XFH) is a **de-facto-standard request** header that records the **original `Host` header the client sent** to the outermost proxy — before any intermediary rewrote it. When a request passes through a reverse proxy, load balancer, or CDN, the [`Host`](../03-Request-Headers/Host.md) header your origin sees is often *not* the public hostname the user typed: the proxy may rewrite `Host` to an internal upstream name (e.g. `app-backend.internal`) while routing. `X-Forwarded-Host` preserves the user-facing hostname (e.g. `www.example.com`) so the origin can build correct absolute URLs, generate proper redirects and links, set cookie domains, and route by the public host. It's the host-carrying member of the forwarding family alongside [`X-Forwarded-For`](./X-Forwarded-For.md) (client IP) and [`X-Forwarded-Proto`](./X-Forwarded-Proto.md) (scheme) — and together those three reconstruct the full original URL: `{X-Forwarded-Proto}://{X-Forwarded-Host}{path}`. Like all forwarding headers it is **client-forgeable** and, because it feeds URL generation, it is a notorious vector for **host-header injection**, **cache poisoning**, and **poisoned password-reset links**. The RFC-standard equivalent is the `host=` parameter of [`Forwarded`](./Forwarded.md).

## What problem does this header solve?

Reverse proxies frequently **rewrite the `Host` header** as part of routing. A proxy might terminate `https://shop.example.com`, then forward to an internal service using `Host: shop-svc.cluster.local` (or strip the port, or use a different vhost name) so the backend's virtual-host routing works. From the origin's perspective, `req.headers.host` is now the *internal* name — but the application often needs the *public* name to function correctly:

- **Absolute URL generation:** redirect `Location`, canonical links, sitemaps, RSS `<link>`, `og:url` meta, and API `self` links must use the public host, not the internal one.
- **Password-reset / email-verification links:** these are built server-side from the host; using the internal host produces links that don't resolve for the user.
- **OAuth/OIDC redirect URIs and webhooks:** callback URLs must be the public host.
- **Multi-tenant routing:** an app serving many customer domains (`acme.saas.com`, `globex.saas.com`) needs the *original* host to pick the tenant.
- **Cookie domain scoping:** setting a cookie for the right domain requires the public host.

`X-Forwarded-Host` preserves that original hostname across the proxy hops so the origin can do all of the above correctly, even when `Host` has been rewritten internally.

## Why was it introduced?

`X-Forwarded-Host` emerged, like its siblings, as an informal convention from reverse proxies and load balancers (Squid, Apache, Nginx, cloud LBs) — a pragmatic response to the reality that proxies rewrite `Host` for routing while backends still need the public hostname. It followed the `X-` naming pattern that RFC 6648 later deprecated for new headers. The concept was standardized as the `host=` parameter of [`Forwarded`](./Forwarded.md) (**RFC 7239, 2014**), but XFH remains widely deployed because it predates and outnumbers the standard. Its existence is a direct consequence of two facts colliding: HTTP routing depends on `Host`, and proxies must sometimes change `Host` to route — so the *original* value needs a separate channel to survive.

## How does it work?

The outermost proxy captures the client's `Host` and places it in `X-Forwarded-Host` before (optionally) rewriting `Host` for internal routing. The origin reads XFH to recover the public hostname.

```mermaid
sequenceDiagram
    participant C as Client
    participant RP as Reverse proxy / CDN
    participant App as Origin app
    C->>RP: GET / (Host: shop.example.com)
    Note over RP: Route internally; rewrite Host.<br/>Preserve original in XFH.
    RP->>App: GET /<br/>Host: shop-svc.internal<br/>X-Forwarded-Host: shop.example.com<br/>X-Forwarded-Proto: https
    Note over App: Public host = shop.example.com (from XFH)<br/>Build https://shop.example.com/... URLs
    App-->>RP: 301 Location: https://shop.example.com/...
    RP-->>C: 301 (correct public URL)
```

- **Browser behavior:** Browsers do **not** send `X-Forwarded-Host` — they send [`Host`](../03-Request-Headers/Host.md). XFH is created by intermediaries.
- **Server behavior:** The origin reads XFH (only from trusted proxies) to determine the public host, then uses it for URL building, routing, and cookies. Frameworks expose it via host helpers (Express `req.hostname` when `trust proxy` is set).
- **Proxy behavior:** The outermost proxy sets XFH to the inbound `Host`; it may rewrite `Host`. It should **overwrite** any client-supplied XFH at the trust boundary.
- **CDN behavior:** CDNs set XFH to the viewer's host and usually preserve the public `Host` too; behavior varies, so check whether your CDN forwards the original `Host` or rewrites it.
- **Reverse proxy behavior:** Nginx sets it via `proxy_set_header X-Forwarded-Host $host` and typically also passes `Host` (`proxy_set_header Host $host`), depending on whether it rewrites the host for the upstream.

## HTTP Request Example

A request where the proxy rewrote `Host` but preserved the public host in XFH:

```http
GET /account HTTP/1.1
Host: app-backend.internal
X-Forwarded-Host: www.example.com
X-Forwarded-Proto: https
X-Forwarded-For: 203.0.113.7
```

A **forged** request aiming to poison a generated link (host-header injection):

```http
POST /password-reset HTTP/1.1
Host: www.example.com
X-Forwarded-Host: attacker.evil.com

{"email":"victim@example.com"}
```

If the app builds the reset link from XFH without validation, the victim receives an email with a link pointing to `attacker.evil.com` — which captures the reset token when clicked.

## HTTP Response Example

`X-Forwarded-Host` is **request-only** — it never appears on responses. Its effect shows up in response headers that embed the host, e.g. a redirect built from the (validated) public host:

```http
HTTP/1.1 301 Moved Permanently
Location: https://www.example.com/account
Set-Cookie: session=abc; Domain=example.com; Secure; HttpOnly
```

## Express.js Example

Express derives `req.hostname` from `X-Forwarded-Host` **only when `trust proxy` is set** — and you must **validate** the host against an allowlist before using it to build links:

```js
const express = require('express');
const app = express();

// 1) trust proxy makes req.hostname honor X-Forwarded-Host. Exact hops/subnets,
//    NEVER `true` — an attacker could otherwise forge the host you build links from.
app.set('trust proxy', 1);

// 2) Allowlist of hostnames you actually serve. This is the core defense against
//    host-header injection / poisoned links.
const ALLOWED_HOSTS = new Set(['www.example.com', 'shop.example.com', 'example.com']);

app.use((req, res, next) => {
  // req.hostname comes from X-Forwarded-Host (trust proxy) or Host otherwise.
  const host = req.hostname;
  if (!ALLOWED_HOSTS.has(host)) {
    // Reject unknown hosts rather than reflect them into URLs/emails.
    return res.status(400).send('Invalid host');
  }
  // Stash a validated public base URL for safe link generation everywhere.
  req.publicBase = `${req.protocol}://${host}`;   // e.g. https://www.example.com
  next();
});

// 3) Build user-facing links ONLY from the validated base — never raw XFH.
app.post('/password-reset', async (req, res) => {
  const token = await createResetToken(req.body.email);
  const link = `${req.publicBase}/reset?token=${token}`; // safe: host was allowlisted
  await sendEmail(req.body.email, link);
  res.json({ ok: true });
});

// 4) Multi-tenant routing by the original public host.
app.use((req, res, next) => {
  req.tenant = tenantForHost(req.hostname); // pick tenant from the public host
  next();
});

app.listen(3000);
```

Why each piece matters: `app.set('trust proxy', 1)` makes `req.hostname` reflect the public host (from XFH) instead of the internal `Host` — without it, your links contain `app-backend.internal`. But trusting XFH is only safe **with the allowlist in step 2**: because XFH is forgeable, reflecting it directly into a password-reset URL (step 3) is the classic account-takeover bug — the attacker sets `X-Forwarded-Host: evil.com`, the victim gets an email linking to the attacker's domain, and clicking it leaks the reset token. Validating against `ALLOWED_HOSTS` and building links only from `req.publicBase` closes that hole. Setting `trust proxy: true` reopens it by trusting any forged XFH.

## Node.js Example

Raw `http` — trust-gate and allowlist yourself:

```js
const http = require('http');

const TRUSTED_PEERS = ['10.0.0.9', '198.51.100.4'];
const ALLOWED_HOSTS = new Set(['www.example.com', 'shop.example.com']);

http.createServer((req, res) => {
  const peer = req.socket.remoteAddress;
  const trusted = TRUSTED_PEERS.includes(peer);

  // Prefer XFH only from a trusted proxy; else fall back to Host.
  const rawHost = (trusted && req.headers['x-forwarded-host'])
    ? req.headers['x-forwarded-host'].split(',')[0].trim()  // leftmost = client's host
    : req.headers['host'];

  const host = (rawHost || '').split(':')[0];               // strip port for comparison

  if (!ALLOWED_HOSTS.has(host)) {
    res.statusCode = 400;
    return res.end('Invalid host');
  }

  const proto = (trusted && req.headers['x-forwarded-proto']) || 'https';
  const base = `${proto}://${host}`;
  res.end(JSON.stringify({ base }));
}).listen(3000);
```

Same discipline: believe XFH only from trusted peers, take the leftmost value, strip the port, and **validate against an allowlist** before it ever enters a generated URL.

## React Example

React never sends or reads `X-Forwarded-Host` — the browser sends [`Host`](../03-Request-Headers/Host.md). The relationship is indirect:

1. **Broken/poisoned links surface in the React UI or emails.** If the server misreads the host (XFH not honored), links (share URLs, canonical tags, API `self` links) contain the *internal* host and break for users. If the server trusts XFH *without validation*, poisoned links (password reset, invites) point to attacker domains — an account-takeover risk that manifests in emails your React app triggered.

2. **SSR / Next.js absolute URLs & multi-tenancy.** Server-rendered React building canonical/`og:` URLs, and multi-tenant apps selecting a tenant by hostname, rely on the forwarded host. Frameworks read `X-Forwarded-Host` (via the platform) to construct the request URL; misconfiguration yields wrong-host canonicals (SEO harm) or wrong-tenant rendering.

3. **You don't set it in fetch/axios** — the browser controls `Host`, and a correct server validates/ignores client-supplied XFH regardless.

## Browser Lifecycle

There is **no browser lifecycle** for `X-Forwarded-Host`. The browser sends `Host`, not XFH; it never reads XFH or exposes it to JS. The header is created at the first proxy that (potentially) rewrites `Host`, capturing the original there, and consumed at the origin. The browser's only role is supplying the `Host` that becomes the XFH value.

## Production Use Cases

- **Absolute URL generation:** redirects, canonical links, sitemaps, feeds, `og:`/`twitter:` meta, API `self`/HATEOAS links — all using the public host.
- **Password-reset / verification / invite links:** built server-side from the (validated) public host.
- **OAuth/OIDC redirect URIs & webhook callback URLs:** must be the public host.
- **Multi-tenant SaaS routing:** select the tenant from the original hostname when the proxy rewrites `Host`.
- **Cookie domain scoping:** set cookies for the correct public domain.
- **Vanity/custom domains:** platforms serving customer domains need the original host to map to the right site.

## Common Mistakes

- **Reflecting XFH into URLs without validation.** The #1 mistake — enables host-header injection, poisoned reset links, and open redirects. Always allowlist.
- **`trust proxy: true` / trusting XFH from any peer.** Makes forgery trivial. Trust only your proxies; validate the value regardless.
- **Not honoring XFH at all.** Links/redirects contain the internal upstream host and break for users.
- **Forgetting the port.** XFH may include a port; strip or handle it consistently when comparing/building URLs (and consider `X-Forwarded-Port`).
- **Using the wrong element on multi-value XFH.** Take the leftmost (client) host.
- **Trusting XFH for security decisions (auth, CSRF origin checks).** Host is not identity; use it for routing/URLs, not authorization.
- **Cache key mismatch.** If responses vary by host but the cache doesn't key on it (or keys on the internal `Host`), you can serve one host's content for another — a cache-poisoning risk.

## Security Considerations

- **Host-header injection → account takeover.** The signature attack: forge `X-Forwarded-Host: attacker.com`, trigger a password reset for a victim; the app emails a reset link on the attacker's domain; when the victim clicks, the token leaks. **Mitigation: never build links from an unvalidated host — allowlist the hosts you serve and reject/normalize everything else.**
- **Web cache poisoning.** If XFH influences a cached response (e.g. an absolute URL baked into HTML) and the cache doesn't key on it, an attacker can poison the cached page for all users with an attacker-controlled host. Don't reflect unkeyed host input into cacheable responses; add [`Vary`](../06-Caching-Headers/Vary.md) / correct cache keys where host truly varies content.
- **Open redirect.** Redirecting to a host taken from XFH without validation is an open redirect. Redirect only to allowlisted hosts.
- **Forgeable — trust boundary matters.** As with all forwarding headers, believe XFH only from your proxies and firewall the origin so the edge can't be bypassed.
- **CSRF/SameSite interplay.** Don't use XFH as an origin/host check for CSRF; use proper [`Origin`](../03-Request-Headers/Origin.md) validation and SameSite cookies.

## Performance Considerations

- **Negligible wire cost;** a short header, compressed under HTTP/2/3.
- **Correctness over speed:** the real impact is avoiding broken links and poisoned caches — failures that are far costlier than the header's bytes.
- **Cache-key implications:** if content genuinely varies by host, the host must be part of the cache key — otherwise you either mis-serve (correctness/security) or fragment the cache unnecessarily. Be deliberate.
- **Multi-tenant routing** by host is cheap (a lookup) but must be consistent across the fleet/CDN.

## Reverse Proxy Considerations

Nginx sets XFH and decides whether to also rewrite `Host`:

```nginx
server {
  server_name www.example.com;

  location / {
    proxy_pass http://app_backend;

    # Preserve the public host for the backend:
    proxy_set_header X-Forwarded-Host $host;        # original client Host
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    # Option A: forward the original Host unchanged (backend sees public host):
    proxy_set_header Host $host;

    # Option B: rewrite Host for internal vhost routing (backend relies on XFH):
    # proxy_set_header Host app-backend.internal;
  }
}
```

Key points: choose deliberately between passing `Host` through (Option A — simplest; the backend can just use `Host`) and rewriting it (Option B — then the backend *must* rely on XFH for the public name). At the **internet-facing edge**, reset/overwrite any client-supplied `X-Forwarded-Host` to the real inbound `$host` so forged values can't survive. `$host` is Nginx's parsed host (from `Host` or the request line), stripped of port.

## CDN Considerations

- **CDNs set `X-Forwarded-Host` to the viewer's host;** many also forward the original `Host`. Verify which your CDN does — some rewrite `Host` to the origin's hostname, making XFH the only source of the public host.
- **Custom/vanity domains:** platforms serving customer domains rely on XFH (or the preserved `Host`) to route to the right site; misconfiguration serves the wrong tenant.
- **Cache poisoning guardrails:** ensure the CDN doesn't cache host-reflected content under a key that ignores the host; prefer not to reflect unvalidated host input into cacheable HTML.
- **Trust only CDN egress ranges** for XFH and firewall the origin against direct access.

## Cloud Deployment Considerations

- **AWS ALB:** preserves the client `Host` by default (and sets `X-Forwarded-*`); if you route by host, the app can often use `Host` directly. Confirm behavior for host-based routing rules.
- **GCP/Azure LBs & App Gateway/Front Door:** may rewrite `Host` to a backend name; then XFH carries the public host — check the specific product's forwarding behavior.
- **API Gateways / meshes:** propagate XFH; multi-tenant setups read it for routing.
- **Managed platforms (Vercel/Netlify):** expose the request host via platform APIs and handle custom domains; use the platform's host helper rather than hand-parsing XFH, and rely on their host validation.
- **Universal rule:** know whether your platform preserves or rewrites `Host`, honor XFH accordingly, and **always allowlist** the hosts you serve before building URLs.

## Debugging

- **curl (injection test):** `curl -H 'X-Forwarded-Host: evil.com' https://your-app/password-reset -d '{"email":"x@y.com"}'` — inspect the generated link/email; if it contains `evil.com`, you have a host-injection bug (missing allowlist).
- **Echo endpoint:** return `req.hostname`, `req.headers.host`, `req.headers['x-forwarded-host']`, and your computed `publicBase` to see what each layer contributes.
- **curl through the real path:** hit the public URL and confirm generated links use the correct public host.
- **Nginx:** log `$host`, `$http_host`, and `$http_x_forwarded_host` to see inbound vs forwarded host and any rewrite.
- **Check emails/links:** trigger a reset in staging and verify the link host is allowlisted and correct.
- **Cache test:** request with a rogue XFH and confirm no cached response reflects it for other users.

## Best Practices

- [ ] Set `trust proxy` to your **exact** topology so `req.hostname` honors XFH — never `true`.
- [ ] **Allowlist** the hostnames you serve; reject/normalize anything else **before** using the host in URLs, emails, redirects, or routing.
- [ ] Build user-facing links only from a **validated** public base (`{proto}://{allowlisted-host}`), never raw XFH.
- [ ] Overwrite client-supplied XFH at the internet-facing edge; trust it only from your proxies.
- [ ] Strip/handle the port; use the leftmost value on multi-value XFH.
- [ ] Don't reflect unvalidated host into **cacheable** responses; key caches on host where content varies.
- [ ] Never use XFH for authentication or CSRF origin checks — use [`Origin`](../03-Request-Headers/Origin.md) + SameSite.
- [ ] Pair with [`X-Forwarded-Proto`](./X-Forwarded-Proto.md) (and port) for full URL reconstruction.
- [ ] Consider [`Forwarded`](./Forwarded.md) (`host=`) for standardized semantics.

## Related Headers

- [Host](../03-Request-Headers/Host.md) — the original header XFH preserves; what proxies may rewrite.
- [X-Forwarded-Proto](./X-Forwarded-Proto.md) — scheme; combined with XFH to build the full absolute URL.
- [X-Forwarded-For](./X-Forwarded-For.md) — client IP; same trust model and `trust proxy` switch.
- [Forwarded](./Forwarded.md) — RFC 7239 standard; `host=` is the XFH equivalent.
- [Origin](../03-Request-Headers/Origin.md) — the correct header for CSRF/CORS origin checks (not XFH).
- [Set-Cookie](../08-Cookies/Set-Cookie.md) — cookie `Domain` scoping depends on the public host.
- [Vary](../06-Caching-Headers/Vary.md) — cache-key correctness when responses vary by host.
- [Proxies Overview](./Proxies-Overview.md) — the forwarding/trust framing.

## Decision Tree

```mermaid
flowchart TD
    A[Need the public hostname?] --> B{Does a proxy rewrite Host?}
    B -- No --> C[Use Host directly<br/>(still validate against allowlist)]
    B -- Yes --> D[Set trust proxy = exact hops/subnets]
    D --> E[Read X-Forwarded-Host (leftmost) via framework]
    E --> F{Host ∈ allowlist?}
    F -- No --> G[Reject / 400 — do NOT reflect it]
    F -- Yes --> H[Build URLs/route/set cookies from it]
    A --> I[NEVER put unvalidated XFH into<br/>links, redirects, or cached HTML]
```

## Mental Model

Think of `X-Forwarded-Host` as the **"deliver replies to this address" label a mailroom keeps when it re-sorts your letter into an internal pouch**. You addressed your letter to the company's public address (`Host: www.example.com`), but the internal mailroom repackages it into an unmarked interoffice envelope routed to a specific department (`Host: app-backend.internal`) — so the clerk who finally opens it can't see the public address you used. The `X-Forwarded-Host` label is the sticky note that preserves "the sender wrote to *www.example.com*," so when the department mails you back (redirects, reset links, confirmations), they use the address you'd actually recognize, not the internal department code. The peril is that this note is written in *pencil by whoever mailed the letter* — a scammer can scribble "reply to *attacker.com*" and, if the department blindly mails your password reset there, your secrets go to the scammer. So the office rule is strict: **only mail replies to addresses on the company's approved list** (allowlist), no matter what the sticky note says — and the front desk erases any suspicious note the moment mail arrives from outside.
