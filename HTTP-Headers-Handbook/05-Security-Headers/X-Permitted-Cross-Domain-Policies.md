# X-Permitted-Cross-Domain-Policies

## Quick Summary

`X-Permitted-Cross-Domain-Policies` is a **response** header (a non-standard convention) that controls whether **Adobe products — historically Flash Player and Adobe Acrobat/PDF — may load "cross-domain policy files"** from your site to grant themselves cross-origin data access. Its most common and recommended value is `none`, which tells these clients "**do not honor any cross-domain policy files on this host**," effectively slamming shut a legacy cross-origin access channel that predates and bypasses CORS. It exists because Adobe's Flash had its own cross-domain permission system based on a `crossdomain.xml` file at your site root; a misconfigured (or attacker-planted) `crossdomain.xml` could let a malicious SWF read your users' authenticated data cross-origin — a serious hole entirely separate from the browser's Same-Origin Policy and CORS. Today, with **Flash dead (end-of-life December 2020)** and this being a legacy concern, the header is mostly a **defense-in-depth / compliance-checklist item**: setting `X-Permitted-Cross-Domain-Policies: none` costs nothing and definitively closes the door. It's set by `helmet` by default. It is *not* something modern apps rely on functionally — it's about *denying* an obsolete but historically-dangerous cross-origin mechanism.

## What problem does this header solve?

Before (and alongside) CORS, Adobe Flash had its **own** cross-origin access model. A SWF (Flash movie) hosted on `evil.com` couldn't normally read data from `yourbank.com` — *unless* `yourbank.com` served a **`crossdomain.xml`** policy file granting permission (e.g. `<allow-access-from domain="*"/>`). Crucially, this bypassed the browser's Same-Origin Policy entirely: it was Flash's runtime, not the browser, enforcing (or not enforcing) access. This created a dangerous, easily-overlooked hole:

- A **too-permissive `crossdomain.xml`** (especially `domain="*"`) let *any* Flash content on the web make credentialed requests to your site and read the responses — a cross-origin data-theft vector orthogonal to CORS.
- An **attacker who could upload a file** to your domain (e.g. a user-content host) might plant a malicious `crossdomain.xml` (or the related `crossdomain` policy referenced by other means), opening access you never intended.
- Adobe Acrobat/PDF and other Adobe runtimes honored similar policy files.

`X-Permitted-Cross-Domain-Policies: none` solves this by instructing Adobe clients to **ignore all such policy files** for your host — meaning even if a `crossdomain.xml` exists (legitimately or maliciously), it won't be honored. It's a blanket "this legacy door is closed" declaration, removing an entire class of cross-origin risk that CORS/SOP don't cover.

## Why was it introduced?

The header comes from **Adobe's cross-domain policy specification** (the same spec that defines `crossdomain.xml`). Adobe recognized that relying solely on a file at the site root was fragile — sites with user-uploaded content or shared hosting couldn't easily prevent a rogue policy file — so they added a **response header** that could override/restrict which policy files clients honor, with `none` meaning "honor no policy files at all." It follows the `X-` naming convention (deprecated for new headers by RFC 6648) and was never an IETF/W3C standard — it's an Adobe-runtime convention. Its relevance has **declined sharply** with Flash's end-of-life (Dec 31, 2020) and browsers removing Flash support; today it's retained in hardening tooling (`helmet` sets `none` by default) as cheap, definitive closure of a legacy hole and to satisfy security scanners/compliance checklists that still flag its absence.

## How does it work?

The header tells Adobe clients which cross-domain policy files (if any) they may honor for the host. Common values:

- **`none`** — honor **no** policy files (recommended; closes the channel).
- **`master-only`** — honor only the master policy file at the site root (`/crossdomain.xml`), ignoring others in subdirectories.
- **`by-content-type`** / **`by-ftp-filename`** — honor policies only for specific content types / FTP names.
- **`all`** — honor all policy files (permissive; avoid).

```mermaid
flowchart TD
    A[Adobe client wants cross-domain access to your host] --> B{X-Permitted-Cross-Domain-Policies?}
    B -- none --> C[Ignore ALL crossdomain.xml → no legacy cross-origin access]
    B -- master-only --> D[Honor only /crossdomain.xml]
    B -- all / absent --> E[Honor policy files → potential legacy hole]
    C --> F[Modern browsers: irrelevant (Flash is dead),<br/>but the door is definitively shut]
```

- **Browser behavior:** Modern browsers **do not use** this header — they have no Flash and don't process `crossdomain.xml`. It targets Adobe runtimes specifically, so in a Flash-free world it has no functional effect on normal browsing. It's purely a signal to (now largely extinct) Adobe clients and a checklist/scanner item.
- **Server behavior:** The origin sets it (typically `none`) as a blanket policy; it may also serve/deny an actual `crossdomain.xml`.
- **Proxy/CDN behavior:** Pass it through; can inject `none` centrally.
- **Reverse proxy behavior:** A convenient place to set `none` site-wide.

## HTTP Request Example

This is a **response** header with no request-side form. The relevant "request" was historically an Adobe client fetching your `crossdomain.xml`:

```http
GET /crossdomain.xml HTTP/1.1
Host: api.example.com
```

## HTTP Response Example

Recommended hardening — deny all policy files:

```http
HTTP/1.1 200 OK
Content-Type: application/json
X-Permitted-Cross-Domain-Policies: none

{"data": "..."}
```

If you must serve a master policy but restrict to it only:

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
X-Permitted-Cross-Domain-Policies: master-only
```

And the actual (legacy) `crossdomain.xml` you'd want to keep *restrictive* if present at all:

```xml
<?xml version="1.0"?>
<cross-domain-policy>
  <!-- If you serve this at all, do NOT use domain="*". Restrict tightly. -->
  <allow-access-from domain="trusted.example.com"/>
</cross-domain-policy>
```

## Express.js Example

```js
const express = require('express');
const app = express();

// 1) helmet sets X-Permitted-Cross-Domain-Policies: none by default.
const helmet = require('helmet');
app.use(helmet()); // includes X-Permitted-Cross-Domain-Policies: none

// 2) Explicit equivalent (if not using helmet):
app.use((req, res, next) => {
  res.set('X-Permitted-Cross-Domain-Policies', 'none'); // close the legacy Adobe channel.
  next();
});

// 3) Actively deny the legacy policy file too (belt-and-suspenders).
app.get('/crossdomain.xml', (req, res) => {
  res.status(404).end();   // don't serve a cross-domain policy at all.
});
// (Also relevant: /clientaccesspolicy.xml for Silverlight — equally obsolete; deny it too.)
app.get('/clientaccesspolicy.xml', (req, res) => res.status(404).end());

app.listen(3000);
```

Why each piece matters: for the vast majority of modern apps, `helmet()` (route 1) already sets `none`, and that's all you need — a zero-cost, definitive closure that also quiets security scanners. Route 2 is the explicit form. Route 3 is genuine defense-in-depth: setting the header **and** ensuring no `crossdomain.xml`/`clientaccesspolicy.xml` (the Silverlight analog) is served — especially important on hosts that accept **user uploads**, where an attacker planting such a file at a predictable path could historically open cross-origin access. Since Flash/Silverlight are dead, the practical impact is minimal, but the cost of closing the door is also nil, so there's no reason not to.

## Node.js Example

Raw `http`:

```js
const http = require('http');

http.createServer((req, res) => {
  // Deny legacy cross-domain policy files outright.
  if (req.url === '/crossdomain.xml' || req.url === '/clientaccesspolicy.xml') {
    res.statusCode = 404;
    return res.end();
  }
  // Blanket-close the Adobe cross-domain channel on every response.
  res.setHeader('X-Permitted-Cross-Domain-Policies', 'none');
  res.setHeader('Content-Type', 'application/json');
  res.end('{"ok":true}');
}).listen(3000);
```

The whole strategy in one place: header `none` + refuse to serve the policy files.

## React Example

React has **no interaction** with this header — it targets Adobe Flash/Acrobat runtimes, which browsers no longer support. There is nothing to do in React code. The only relevance to a React developer is:

1. **It's a server-config hardening item**, not a frontend concern. Your backend/host should send `X-Permitted-Cross-Domain-Policies: none` as part of a standard security-header set (e.g. via `helmet` on an Express API, or host config for a static React deploy).

2. **Security scanners flag its absence.** If a pentest/scanner report on your React app's hosting dings "missing X-Permitted-Cross-Domain-Policies," the fix is a one-line server/host header addition — not a code change.

3. **User-upload hosts** (if your React app has an assets/upload domain) should both send `none` and ensure no `crossdomain.xml` can be uploaded to a served path.

## Browser Lifecycle

There is effectively **no modern browser lifecycle** for this header. Contemporary browsers don't run Flash and don't consume cross-domain policy files, so the header has no effect on normal page/resource loading. Its "lifecycle" was entirely within the (now-defunct) Adobe Flash/Acrobat runtimes: those clients, before making a cross-origin request, would check the header and any `crossdomain.xml` to decide whether access was permitted. In today's web it's a static declaration that closes a legacy door no live browser walks through — meaningful only to residual Adobe tooling and to security auditors.

## Production Use Cases

- **Security-header baseline / compliance:** including `none` to satisfy scanners, pentests, and hardening checklists (CIS, OWASP secure-headers).
- **Defense-in-depth on user-upload hosts:** preventing a planted `crossdomain.xml` from opening legacy cross-origin access.
- **Legacy environment hardening:** any environment where residual Adobe clients might exist.
- **Blanket API hardening:** setting `none` across all API responses as a matter of policy.

## Common Mistakes

- **Thinking it affects CORS/modern browsers.** It doesn't — it's Adobe-runtime-specific and largely moot post-Flash. Don't rely on it for any modern cross-origin control (that's [CORS](../07-CORS/CORS-Overview.md)/[CORP](./Cross-Origin-Resource-Policy.md)).
- **Setting the header but still serving a permissive `crossdomain.xml`.** For residual clients, `none` in the header should be paired with not serving (or restricting) the policy file.
- **Using `all` / `domain="*"`.** The permissive settings are the historical vulnerability; avoid.
- **Skipping it entirely on user-content hosts.** Cheap to add; closes a real (if legacy) upload-based vector.
- **Overvaluing it.** It's a minor, legacy checklist item — don't prioritize it over [CSP](./Content-Security-Policy.md), [HSTS](./Strict-Transport-Security.md), CORS correctness.

## Security Considerations

- **Closes a legacy cross-origin channel** that operated *outside* the Same-Origin Policy/CORS — historically exploitable via permissive or attacker-planted policy files.
- **Most relevant on upload/user-content hosts:** where an attacker could place a `crossdomain.xml` at a served path; `none` neutralizes it for compliant Adobe clients (and you should also refuse to serve such files).
- **Defense-in-depth, low current impact:** with Flash EOL, real-world exposure is minimal, but the mitigation cost is zero.
- **Not a substitute for modern controls:** cross-origin protection today is [CORS](../07-CORS/CORS-Overview.md), [CORP](./Cross-Origin-Resource-Policy.md), [COOP](./Cross-Origin-Opener-Policy.md)/[COEP](./Cross-Origin-Embedder-Policy.md), and [CSP](./Content-Security-Policy.md) — this header addresses only the Adobe legacy vector.
- **Silverlight analog:** `clientaccesspolicy.xml` was Microsoft Silverlight's equivalent; deny it too for completeness (also obsolete).

## Performance Considerations

- **Zero meaningful performance impact** — a tiny static header with no processing cost on modern clients.
- **No caching interaction of note.**
- **The only "cost"** is a few bytes per response (negligible, compressed under HTTP/2/3), which is why there's no reason to omit it in a hardening baseline.

## Reverse Proxy Considerations

Nginx setting `none` and denying policy files:

```nginx
server {
  add_header X-Permitted-Cross-Domain-Policies "none" always;

  # Refuse to serve legacy cross-domain policy files.
  location = /crossdomain.xml { return 404; }
  location = /clientaccesspolicy.xml { return 404; }

  location / { proxy_pass http://app_upstream; }
}
```

Key points: a central `add_header ... always` applies the policy everywhere; explicitly 404-ing the policy files closes the door even for residual clients. This is a common inclusion in Nginx security-header templates.

## CDN Considerations

- **Edge injection:** CDNs can add `X-Permitted-Cross-Domain-Policies: none` across a property via header rules.
- **Deny policy files at the edge:** block `/crossdomain.xml` and `/clientaccesspolicy.xml` if you want defense-in-depth on CDN-served content (esp. user uploads).
- **Pass-through:** CDNs forward the header; ensure it's set consistently.
- **Low priority:** given Flash's demise, most CDNs treat this as an optional hardening add-on.

## Cloud Deployment Considerations

- **Managed hosts (Vercel/Netlify):** add via `_headers`/config as part of a security-header set.
- **Object storage / upload buckets:** ensure buckets don't serve a permissive `crossdomain.xml`; set the header where the platform allows.
- **API Gateways/LBs:** inject `none` centrally in a standard header policy.
- **Compliance tooling:** many cloud security posture scanners flag its absence — including it is a quick win.

## Debugging

- **curl:** `curl -sD - -o /dev/null https://host/ | grep -i x-permitted-cross-domain-policies` → confirm `none`.
- **Check for policy files:** `curl -i https://host/crossdomain.xml` and `.../clientaccesspolicy.xml` → should be `404`/absent (or tightly restricted if present).
- **Security scanners:** tools like Mozilla Observatory, securityheaders.com, and pentest suites report on this header.
- **No browser DevTools interaction:** since modern browsers ignore it, there's nothing to observe in DevTools beyond the header's presence.
- **Header baseline audit:** include it in your automated header-presence tests.

## Best Practices

- [ ] Set `X-Permitted-Cross-Domain-Policies: none` as part of your standard security-header baseline (or use `helmet`, which does).
- [ ] Also **refuse to serve** `crossdomain.xml` (and `clientaccesspolicy.xml`) unless you have a specific legacy need.
- [ ] **Never** use permissive policy files (`domain="*"`) or `all`.
- [ ] Pay special attention on **user-upload/content hosts** (planted-file risk).
- [ ] Treat it as a **low-priority, zero-cost** hardening/compliance item — don't let it distract from [CSP](./Content-Security-Policy.md)/[HSTS](./Strict-Transport-Security.md)/CORS.
- [ ] Apply it consistently across the property (proxy/CDN/edge injection).
- [ ] Include it in automated header-baseline tests.

## Related Headers

- [Content-Security-Policy](./Content-Security-Policy.md) — the modern, primary control over what a page may load/execute.
- [Cross-Origin-Resource-Policy](./Cross-Origin-Resource-Policy.md) — modern control over who may load a resource cross-origin.
- [Access-Control-Allow-Origin](../07-CORS/Access-Control-Allow-Origin.md) — the modern cross-origin *reading* mechanism (CORS) that superseded Flash's model.
- [X-Content-Type-Options](./X-Content-Type-Options.md) / [X-Frame-Options](./X-Frame-Options.md) — fellow legacy-ish `X-` hardening headers in the same baseline.
- [Strict-Transport-Security](./Strict-Transport-Security.md) — higher-priority transport hardening.
- [Security Hardening](../21-Best-Practices/Security-Hardening.md) — where this fits in a full checklist.

## Decision Tree

```mermaid
flowchart TD
    A[Building a security-header baseline?] --> B[Set X-Permitted-Cross-Domain-Policies: none]
    B --> C{Do you serve user uploads / shared content?}
    C -- Yes --> D[Also 404 crossdomain.xml + clientaccesspolicy.xml]
    C -- No --> E[Header alone is sufficient (defense-in-depth)]
    A --> F{Need real cross-origin control?}
    F -- Yes --> G[Use CORS / CORP / COOP / COEP / CSP — NOT this header]
```

## Mental Model

Think of `X-Permitted-Cross-Domain-Policies: none` as **bricking up a side door that a demolished wing of the building used to have.** Long ago, your building had a separate Adobe-branded entrance with its own guard who checked a *different* guest list (`crossdomain.xml`) than the main lobby's security (the browser's Same-Origin Policy / CORS) — and if that side-door guest list was sloppy (or someone slipped a forged list under the door), strangers could wander into secure areas the main lobby would never have allowed. That entire wing has since been demolished (Flash is dead), so almost nobody uses the side door anymore. But the *doorway* technically still exists in the blueprints, and a thorough safety inspector (security scanner) will note it. Bricking it up (`none`) costs a single afternoon and guarantees that even if some ancient key still floats around, the door simply doesn't open. You wouldn't *prioritize* this over reinforcing the main entrance ([CSP](./Content-Security-Policy.md), [HSTS](./Strict-Transport-Security.md), CORS), but since sealing it is nearly free and permanently removes a historical break-in route, there's no reason to leave it open.
