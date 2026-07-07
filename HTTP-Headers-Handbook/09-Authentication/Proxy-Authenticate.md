# Proxy-Authenticate

## Quick Summary

`Proxy-Authenticate` is a **response** header sent by a **proxy** (not the origin server) alongside a **`407 Proxy Authentication Required`** status, telling the client which authentication scheme(s) it must use to authenticate **to the proxy itself** — e.g. `Proxy-Authenticate: Basic realm="Corp Proxy"`. It is the **proxy-tier mirror** of the origin's [`WWW-Authenticate`](./WWW-Authenticate.md): where `WWW-Authenticate` + `401` challenges the client to authenticate to the *origin server*, `Proxy-Authenticate` + `407` challenges the client to authenticate to an *intermediary proxy* it must pass through. The client answers with [`Proxy-Authorization`](./Proxy-Authorization.md) (the proxy-tier mirror of [`Authorization`](./Authorization.md)). It is a **hop-by-hop** header — it applies to the connection between the client and *that specific proxy*, not end-to-end — so it must not be forwarded through to other hops. Its classic use is **corporate/forward proxies** that require employees to authenticate before accessing the internet, and it's uncommon in typical web-app development (where authentication is at the origin), but essential to understand for enterprise networks, egress proxies, and debugging `407`s.

## What problem does this header solve?

Sometimes the entity that needs to authenticate a client isn't the destination server but an **intermediary the client must go through** — most commonly a **corporate forward proxy** that controls and audits outbound internet access. The organization wants to say "before I relay your request to the internet, prove *you* (the employee/user) are authorized to use this proxy." The origin-server auth mechanism ([`WWW-Authenticate`](./WWW-Authenticate.md)/`401`/[`Authorization`](./Authorization.md)) can't express this — it's about authenticating to the *destination*, and those credentials would be forwarded to the origin, which is wrong: proxy credentials should stay between the client and the proxy.

`Proxy-Authenticate` (with `407` and [`Proxy-Authorization`](./Proxy-Authorization.md)) solves this by creating a **parallel, proxy-scoped authentication channel** that is:
- **distinct from origin auth** (a request can require *both* — authenticate to the proxy *and* to the origin), and
- **hop-by-hop** (the proxy credentials are consumed by the proxy and not leaked onward to the origin or other hops).

This lets intermediaries enforce access control on the *use of the proxy itself* without interfering with, or leaking into, the client↔origin authentication.

## Why was it introduced?

`Proxy-Authenticate` has been part of HTTP since **HTTP/1.1 (RFC 2068, 1997; RFC 2616, 1999)**, specified today in **RFC 9110 §11.7.1 (2022)**, alongside the `407` status and [`Proxy-Authorization`](./Proxy-Authorization.md). It was introduced because HTTP/1.1 formalized the proxy/gateway model and needed an authentication mechanism scoped to intermediaries — separate from origin authentication — to support the era's dominant use case: **forward proxies** (corporate gateways, ISP proxies, caching proxies) that gated internet access. Making it **hop-by-hop** was deliberate and critical: proxy credentials must not travel to the origin (that would leak them and confuse origin auth), so `Proxy-Authenticate`/`Proxy-Authorization` are explicitly connection-scoped and stripped between hops. It uses the **same authentication-scheme framework** as [`WWW-Authenticate`](./WWW-Authenticate.md) (Basic, Digest, Negotiate/NTLM, Bearer, etc.), just aimed at the proxy tier.

## How does it work?

When a client's request reaches a proxy that requires authentication and the client hasn't provided valid [`Proxy-Authorization`](./Proxy-Authorization.md), the proxy responds **`407 Proxy Authentication Required`** with `Proxy-Authenticate` describing the scheme/realm. The client re-sends the request with [`Proxy-Authorization`](./Proxy-Authorization.md) credentials for the proxy.

```mermaid
sequenceDiagram
    participant C as Client
    participant P as Forward proxy (requires auth)
    participant O as Origin
    C->>P: GET http://example.com/ (no proxy creds)
    P-->>C: 407 Proxy Authentication Required<br/>Proxy-Authenticate: Basic realm="Corp Proxy"
    Note over C: Obtain/attach proxy credentials
    C->>P: GET http://example.com/<br/>Proxy-Authorization: Basic ZW1wOnB3
    Note over P: Validate proxy creds → relay to origin<br/>(Proxy-Authorization NOT forwarded to origin)
    P->>O: GET / (client's origin auth, if any, passes through)
    O-->>P: 200 OK
    P-->>C: 200 OK
```

- **Browser behavior:** On a `407`, the browser typically shows a **proxy authentication dialog** (separate from a site login), obtains credentials, and resends with [`Proxy-Authorization`](./Proxy-Authorization.md). It handles this at the network layer, largely invisibly to page JS. (`fetch`/XHR generally can't read or set proxy auth.)
- **Server (origin) behavior:** The origin never sees `Proxy-Authenticate`/[`Proxy-Authorization`](./Proxy-Authorization.md) (hop-by-hop, stripped by the proxy) and uses [`WWW-Authenticate`](./WWW-Authenticate.md)/[`Authorization`](./Authorization.md) for its own auth.
- **Proxy behavior:** This is its home — the proxy issues the `407` + `Proxy-Authenticate`, validates [`Proxy-Authorization`](./Proxy-Authorization.md), and **must not forward** either header onward.
- **CDN/reverse proxy behavior:** Reverse proxies usually use origin-style auth ([`WWW-Authenticate`](./WWW-Authenticate.md)/`401`) rather than `407`; `Proxy-Authenticate`/`407` is chiefly a **forward-proxy** concern. A reverse proxy *could* use it, but it's unusual.

## HTTP Request Example

The client's initial request (via the proxy) without proxy credentials, and the authenticated retry:

```http
GET http://example.com/ HTTP/1.1
Host: example.com
```

```http
GET http://example.com/ HTTP/1.1
Host: example.com
Proxy-Authorization: Basic ZW1wbG95ZWU6c2VjcmV0
```

## HTTP Response Example

The proxy challenge (Basic):

```http
HTTP/1.1 407 Proxy Authentication Required
Proxy-Authenticate: Basic realm="Corp Proxy"
Content-Length: 0
```

A stronger scheme (Negotiate/NTLM, common in Windows enterprise environments):

```http
HTTP/1.1 407 Proxy Authentication Required
Proxy-Authenticate: Negotiate
Proxy-Authenticate: NTLM
```

Digest challenge (with nonce, like `WWW-Authenticate` Digest):

```http
HTTP/1.1 407 Proxy Authentication Required
Proxy-Authenticate: Digest realm="Corp Proxy", qop="auth", nonce="dcd98b7102dd2f0e", opaque="5ccc069c403ebaf9"
```

## Express.js Example

Express apps are almost always **origin servers**, so they use [`WWW-Authenticate`](./WWW-Authenticate.md)/`401`, **not** `Proxy-Authenticate`/`407`. You'd only emit `407` if you were *building a forward proxy* — a rare case:

```js
const express = require('express');
const app = express();

// NOTE: This is what a FORWARD PROXY would do — NOT a normal app server.
// A normal origin app should use 401 + WWW-Authenticate instead (see WWW-Authenticate).

function checkProxyCredentials(header) {
  if (!header || !header.startsWith('Basic ')) return false;
  const [user, pass] = Buffer.from(header.slice(6), 'base64').toString().split(':');
  return validateProxyUser(user, pass);
}

app.use((req, res, next) => {
  // A forward proxy authenticating the CLIENT to itself:
  if (!checkProxyCredentials(req.headers['proxy-authorization'])) {
    // 407 + Proxy-Authenticate challenges the client for PROXY credentials.
    res.set('Proxy-Authenticate', 'Basic realm="Corp Proxy"');
    return res.status(407).end();
  }
  // IMPORTANT: when relaying upstream, do NOT forward Proxy-Authorization
  // (hop-by-hop). Strip it before proxying to the origin.
  delete req.headers['proxy-authorization'];
  next(); // ...then relay the request upstream.
});

app.listen(3128); // conventional proxy port

// For a NORMAL origin app, you'd instead do:
// res.set('WWW-Authenticate', 'Bearer realm="api"').status(401).end();
```

Why each piece matters: the big teaching point is the **407-vs-401 distinction** — a normal Express *origin* app should virtually never send `407`/`Proxy-Authenticate`; that's for authenticating a client *to a proxy*. If you *are* the proxy, the critical correctness rule is **stripping `Proxy-Authorization` before relaying upstream** (`delete req.headers['proxy-authorization']`) — because it's hop-by-hop, forwarding it would leak proxy credentials to the origin. The scheme framework (Basic/Digest/Negotiate) is identical to [`WWW-Authenticate`](./WWW-Authenticate.md); only the tier (proxy vs origin) and the status (`407` vs `401`) differ.

## Node.js Example

Raw `http` — both sides (a minimal proxy issuing `407`, and a client responding):

```js
const http = require('http');

// Minimal forward proxy requiring Basic proxy auth.
http.createServer((req, res) => {
  const cred = req.headers['proxy-authorization'];
  if (!cred || !validate(cred)) {
    res.writeHead(407, { 'Proxy-Authenticate': 'Basic realm="Corp Proxy"' });
    return res.end();
  }
  // Strip hop-by-hop proxy auth before relaying upstream (not shown).
  delete req.headers['proxy-authorization'];
  // ... proxy the request to the origin ...
  res.writeHead(200); res.end('relayed');
}).listen(3128);

// A client sending proxy credentials (e.g. via an agent configured for the proxy):
const proxyAuth = 'Basic ' + Buffer.from('employee:secret').toString('base64');
http.get({
  host: 'proxy.corp.local', port: 3128, path: 'http://example.com/',
  headers: { 'Proxy-Authorization': proxyAuth },
}, (res) => { res.resume(); });
```

The essentials: proxy issues `407` + `Proxy-Authenticate`; client resends with [`Proxy-Authorization`](./Proxy-Authorization.md); proxy validates and **strips** it before relaying.

## React Example

React has **no direct interaction** with `Proxy-Authenticate`:

1. **Proxy auth is handled by the browser/OS, not page JS.** When a user is behind an authenticating corporate proxy, the *browser* handles the `407` challenge (often via an OS/browser proxy dialog or integrated Windows auth), transparently to your React app. Your `fetch`/`axios` code neither sees the `407` nor sets [`Proxy-Authorization`](./Proxy-Authorization.md) (it's a [forbidden header](../02-Core-Concepts/Forbidden-and-Restricted-Headers.md) for `fetch`).

2. **You cannot read/set it.** `fetch` can't access proxy-auth headers; the network stack owns them. So there's nothing for React to do.

3. **Symptom awareness:** if enterprise users report failures that look like blocked requests, a corporate proxy requiring auth (returning `407`) could be involved — but the fix is network/proxy configuration, not React code. Your app's *own* authentication uses [`Authorization`](./Authorization.md) + `401`, which is entirely separate.

In short: for React, proxy authentication is invisible plumbing between the user's machine and their network's proxy.

## Browser Lifecycle

1. A request routed through an authenticating forward proxy without valid [`Proxy-Authorization`](./Proxy-Authorization.md) gets a **`407`** with `Proxy-Authenticate`.
2. The browser (or OS network layer) obtains proxy credentials — via a **proxy auth dialog**, stored credentials, or integrated auth (Kerberos/NTLM) — separate from any site login.
3. It resends the request with [`Proxy-Authorization`](./Proxy-Authorization.md); the proxy validates and relays.
4. The proxy **strips** the proxy-auth headers before forwarding to the origin (hop-by-hop).
5. Page JS is largely unaware; `fetch`/XHR can't read the `407` challenge or set proxy auth.
6. Origin authentication ([`401`](./WWW-Authenticate.md)) is a **separate** challenge that may also occur.

## Production Use Cases

- **Corporate forward proxies:** authenticating employees before allowing outbound internet access (the classic case).
- **Egress/filtering proxies:** enforcing per-user access control and auditing on outbound traffic.
- **ISP/carrier proxies:** (historically) authenticating subscribers.
- **Enterprise integrated auth:** Negotiate/Kerberos/NTLM proxy auth in Windows domains.
- **CI/build/egress control:** requiring credentials for outbound requests from build environments through a controlled proxy.
- **Debugging enterprise connectivity:** understanding `407`s when apps/tools fail behind an authenticating proxy.

## Common Mistakes

- **Confusing `407`/`Proxy-Authenticate` with `401`/[`WWW-Authenticate`](./WWW-Authenticate.md).** `407` = authenticate to the **proxy**; `401` = authenticate to the **origin**. A normal app should use `401`.
- **Forwarding [`Proxy-Authorization`](./Proxy-Authorization.md) upstream.** It's hop-by-hop; a proxy must **strip** it before relaying, or it leaks proxy credentials to the origin.
- **Trying to set proxy auth from browser JS.** Forbidden/ignored; the network layer handles it.
- **Using `407` at an origin/reverse proxy for app auth.** Wrong semantics; use `401`.
- **Assuming the origin sees proxy auth.** It doesn't (stripped).
- **Not supporting the scheme the environment needs.** Enterprise proxies often need Negotiate/NTLM, not Basic; using Basic over plaintext is insecure.
- **Basic proxy auth without TLS to the proxy.** Base64 is not encryption — credentials are exposed unless the proxy connection is secured (e.g. HTTPS proxy / tunnel).

## Security Considerations

- **Hop-by-hop stripping is a security requirement.** [`Proxy-Authorization`](./Proxy-Authorization.md) must never be forwarded to the origin or other hops — leaking it exposes proxy credentials. Ensure your proxy strips it.
- **Basic scheme is base64, not encryption.** `Proxy-Authenticate: Basic` credentials are trivially decoded; only use it over a **secured** proxy connection (HTTPS proxy / TLS tunnel), or prefer Digest/Negotiate.
- **Credential prompting/phishing:** a `407` can trigger a system credential prompt; malicious proxies (MITM) could harvest credentials. Trust only proxies your organization controls; secure the client↔proxy channel.
- **Separate from origin auth:** don't reuse origin credentials for proxy auth or vice-versa; they're different trust domains.
- **Enumeration/realm disclosure:** the `realm` reveals which proxy is challenging — usually benign but avoid leaking internal names unnecessarily.
- **Replay/nonce:** Digest/Negotiate provide replay protection; Basic does not — another reason to avoid Basic.

## Performance Considerations

- **Extra round-trip on first auth:** the initial `407` + retry adds a round-trip; browsers typically cache proxy credentials for the session so it's a one-time cost per connection/session.
- **Per-connection auth (NTLM/Negotiate):** connection-oriented schemes may re-authenticate per connection, adding overhead; keep-alive reduces this.
- **Negligible header cost;** the impact is the auth round-trip, not bytes.
- **Not an app-layer concern** for most services — it's network infrastructure.

## Reverse Proxy Considerations

Reverse proxies (Nginx, etc.) typically authenticate clients with **origin-style** [`WWW-Authenticate`](./WWW-Authenticate.md)/`401`, not `407`. `Proxy-Authenticate`/`407` is a **forward-proxy** mechanism. If Nginx acts as a forwarding proxy needing client auth, that's unusual; more commonly:

```nginx
# Reverse proxy authenticating the CLIENT to the ORIGIN uses 401, not 407:
server {
  location /secure/ {
    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/.htpasswd;   # emits WWW-Authenticate + 401
    proxy_pass http://app_upstream;
  }
}
# And when proxying UPSTREAM, hop-by-hop headers (incl. Proxy-Authorization) are
# managed per hop; do not forward client proxy credentials to the origin.
```

Key points: use `401`/[`WWW-Authenticate`](./WWW-Authenticate.md) for reverse-proxy/origin auth. Reserve `407`/`Proxy-Authenticate` for genuine forward-proxy scenarios, and always strip hop-by-hop proxy-auth headers when relaying.

## CDN Considerations

- **CDNs act as reverse proxies** and use origin-style auth ([`401`](./WWW-Authenticate.md)), **not** `407`. You'll rarely see `Proxy-Authenticate` from a CDN.
- **Client-to-CDN** authentication (if any) is typically token/cookie/origin-auth based, not proxy auth.
- **Forward-proxy `407`** happens on the *client's* network side (corporate proxy), independent of the CDN.
- **Debugging:** a `407` in front of your CDN traffic usually indicates the *client's* egress proxy, not your infrastructure.

## Cloud Deployment Considerations

- **Egress proxies in cloud/enterprise networks:** outbound traffic from VPCs/build agents may traverse an authenticating forward proxy issuing `407`; configure clients (and `HTTPS_PROXY`/`http_proxy` env vars with credentials) accordingly.
- **Service-to-service:** cloud APIs use token auth ([`Authorization`](./Authorization.md)), not proxy auth; `407` there indicates a misrouted request through an auth proxy.
- **CI/CD:** build systems behind egress proxies need proxy credentials (often via env vars) to reach external registries/APIs.
- **Managed reverse proxies/LBs:** authenticate with origin-style `401`, not `407`.

## Debugging

- **curl (through a proxy):** `curl -x http://proxy.corp:3128 -U employee:secret http://example.com/` — `-x` sets the proxy, `-U` supplies proxy credentials; without `-U` you'll get a `407` and see `Proxy-Authenticate`.
- **Inspect the challenge:** `curl -x http://proxy.corp:3128 -v http://example.com/` shows the `407` and `Proxy-Authenticate` scheme/realm.
- **Env vars:** set `HTTP_PROXY`/`HTTPS_PROXY` (with `user:pass@host`) for tools that honor them; confirm the tool sends [`Proxy-Authorization`](./Proxy-Authorization.md).
- **Browser:** a `407` triggers a proxy auth dialog; DevTools shows the status but page JS can't read the proxy headers.
- **Verify stripping:** if you operate the proxy, confirm [`Proxy-Authorization`](./Proxy-Authorization.md) is not present in the request the origin receives.
- **Distinguish from `401`:** a `407` means proxy auth; a `401` means origin auth — check which one you're getting.

## Best Practices

- [ ] Use `Proxy-Authenticate`/`407` **only** for authenticating a client **to a forward proxy** — use [`WWW-Authenticate`](./WWW-Authenticate.md)/`401` for origin/reverse-proxy auth.
- [ ] **Strip** [`Proxy-Authorization`](./Proxy-Authorization.md) before relaying upstream (hop-by-hop) — never leak proxy credentials to the origin.
- [ ] Prefer strong schemes (Digest/Negotiate/Kerberos) over **Basic**; if Basic, secure the client↔proxy channel with TLS.
- [ ] Support the scheme(s) your environment requires (often Negotiate/NTLM in enterprises).
- [ ] Don't reuse origin credentials for proxy auth (separate trust domains).
- [ ] Don't attempt to handle proxy auth in browser JS — it's network-layer.
- [ ] Configure egress-proxy credentials via env vars (`HTTPS_PROXY`) for tools/CI behind authenticating proxies.
- [ ] Recognize `407` in debugging as a **proxy** (usually client-side network) concern.

## Related Headers

- [Proxy-Authorization](./Proxy-Authorization.md) — the request header the client sends in response to this challenge (proxy-tier mirror of [`Authorization`](./Authorization.md)).
- [WWW-Authenticate](./WWW-Authenticate.md) — the **origin-server** counterpart (`401`); same scheme framework, different tier.
- [Authorization](./Authorization.md) — origin-tier credentials; contrast with `Proxy-Authorization`.
- [Authentication Overview](./Authentication-Overview.md) — the full auth model (schemes, flows).
- [Connection](../03-Request-Headers/Connection.md) — lists hop-by-hop headers; proxy-auth headers are hop-by-hop.
- [End-to-End vs Hop-by-Hop Headers](../01-Introduction/End-to-End-vs-Hop-by-Hop-Headers.md) — why `Proxy-Authenticate`/`Proxy-Authorization` are not forwarded.
- [Proxies Overview](../14-Proxies/Proxies-Overview.md) — the intermediary model this operates within.

## Decision Tree

```mermaid
flowchart TD
    A[Need to authenticate a client?] --> B{To the ORIGIN or to a PROXY?}
    B -- Origin/reverse proxy --> C[Use 401 + WWW-Authenticate<br/>+ Authorization]
    B -- Forward proxy (client must pass through) --> D[Use 407 + Proxy-Authenticate<br/>+ Proxy-Authorization]
    D --> E[Strip Proxy-Authorization before relaying upstream]
    D --> F{Scheme?}
    F -- Enterprise/Windows --> G[Negotiate / NTLM / Kerberos]
    F -- Basic --> H[Only over a secured proxy channel]
```

## Mental Model

Think of `Proxy-Authenticate`/`407` as the **security checkpoint at your office building's exit that you must clear before you're even allowed to leave for an errand**, versus [`WWW-Authenticate`](./WWW-Authenticate.md)/`401`, which is the **lock on the door of the shop you're going to visit.** The building guard (forward proxy) stops you on the way out: "Show your *employee badge* before I let you through to the outside world" (`407` + `Proxy-Authenticate`); you flash your badge (`Proxy-Authorization`) and are let out. That badge is strictly for the *building's* guard — the shop you visit never sees it, and shouldn't (hop-by-hop stripping); flashing your employee badge at the shop would be both pointless and a security leak. Once outside, the shop may have *its own* lock requiring a *different* key — your membership card for *that shop* (`401` + `WWW-Authenticate` → [`Authorization`](./Authorization.md)). Two independent checkpoints, two independent credentials: one proves you may *use the exit* (the proxy), the other proves you may *enter the destination* (the origin). Most app developers only ever deal with the shop's lock; the building exit guard is the enterprise-network's concern.
