# Access-Control-Allow-Origin

## Quick Summary

`Access-Control-Allow-Origin` (ACAO) is the **response** header a server sends to tell the browser *which origin is allowed to read this cross-origin response*. It is the linchpin of CORS: its presence and value are what the browser checks before handing a cross-origin response to JavaScript. Its value is exactly one of two things — the literal wildcard `*` (any origin, anonymous only), or a **single** serialized origin like `https://app.example.com` (never a list, never with a path). The server sets it; the browser enforces it. Everything else in CORS orbits this header, and getting its value wrong is the source of most "CORS errors" in the console.

## What problem does this header solve?

The [Same-Origin Policy](./CORS-Overview.md) forbids a script on origin A from reading a response from origin B. That's the safe default, but it's too strict for the real world: `app.example.com` legitimately needs to call `api.example.com`; a public data API wants to be consumable by any web app; a widget embedded across many sites needs controlled access. ACAO is the server's mechanism to carve a precise exception into the Same-Origin Policy: *"I permit exactly this origin (or everyone) to read my responses in the browser."* Without ACAO, the only cross-origin data-sharing tools were JSONP (an RCE hazard) and server-side proxies (latency + infra). ACAO replaced those with a single, auditable, per-response permission grant.

## Why was it introduced?

ACAO is the original and central header of the CORS specification (W3C CORS, 2009-2014, later absorbed into the WHATWG **Fetch Standard**; the origin concept itself is formalized in RFC 6454). Before it, cross-origin reads were impossible without hacks. The design deliberately made ACAO an **opt-in, per-response, server-controlled** value so that legacy servers — which emit no such header — remain locked down by default. A server that knows nothing about CORS is automatically safe: with no ACAO, the browser blocks every cross-origin read. You only become reachable cross-origin by *choosing* to emit this header, which is exactly the fail-closed posture a security boundary should have.

## How does it work?

- **Browser behavior:** After receiving a cross-origin response, the browser reads ACAO. If the value is `*` (and the request is not credentialed) or exactly equals the request's [`Origin`](../03-Request-Headers/Origin.md), the browser allows the calling script to read the response. Otherwise it **discards the response** (the network fetch already happened) and rejects the promise with a `TypeError`. For preflighted requests, the browser checks ACAO on *both* the `OPTIONS` preflight response and the actual response. The browser does string matching — `https://app.example.com` ≠ `https://app.example.com/` (trailing slash) ≠ `http://app.example.com`. There is no fuzzy matching, no subdomain wildcarding at the protocol level.
- **Server behavior:** The server decides the value. Two strategies: emit a static `*` (public, anonymous APIs), or **dynamically reflect** the request's `Origin` after checking it against an allowlist (everything credentialed or private). Reflection is the norm because a single server usually serves multiple known front-ends and must support credentials, which forbids `*`.
- **Proxy behavior:** ACAO is an end-to-end response header; forward proxies pass it through. A proxy that strips or rewrites response headers can break CORS invisibly.
- **CDN behavior:** CDNs cache responses. If ACAO is dynamically reflected per-origin, the CDN **must** vary its cache key on the `Origin` request header (via [`Vary: Origin`](../06-Caching-Headers/Vary.md)) or it will serve origin A's ACAO to origin B and cause intermittent, maddening CORS failures. Many CDNs need explicit configuration to include `Origin` in the cache key.
- **Reverse proxy behavior:** Nginx/HAProxy/Envoy often *add* ACAO on behalf of the origin, or the origin sets it and the proxy passes it. A frequent bug: the proxy adds ACAO on 2xx via `add_header` but not on error responses, so failures surface as CORS errors instead of the real 500.

## HTTP Request Example

```http
GET /api/products HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
Accept: application/json
```

The browser attached `Origin` automatically because this is cross-origin. The script cannot set or forge `Origin` — it is a [forbidden header](../03-Request-Headers/Origin.md).

## HTTP Response Example

```http
HTTP/1.1 200 OK
Content-Type: application/json
Access-Control-Allow-Origin: https://app.example.com
Vary: Origin
Content-Length: 512

[{"id":1,"name":"Widget"}]
```

`Access-Control-Allow-Origin` echoes the requesting origin (dynamic reflection), and `Vary: Origin` tells every cache that this response is origin-specific and must not be reused across origins.

## Express.js Example

The production pattern is an **allowlist with dynamic reflection**, using the `cors` middleware so `Vary`, preflight, and credentials are handled correctly:

```js
const express = require('express');
const cors = require('cors');
const app = express();

// The exact origins your front-ends are served from. Exact strings — scheme+host+port.
const ALLOWLIST = new Set([
  'https://app.example.com',
  'https://admin.example.com',
  'http://localhost:3000',        // local dev; drop in prod builds
]);

const corsOptions = {
  origin(origin, callback) {
    // `origin` is undefined for same-origin requests and for non-browser clients
    // (curl, server-to-server) that send no Origin header. Allow those through —
    // CORS is irrelevant to them and blocking would break health checks / SSR.
    if (!origin) return callback(null, true);

    // Exact-match against the allowlist. On match, `cors` reflects THIS origin
    // into Access-Control-Allow-Origin (never `*`), which is required for credentials.
    if (ALLOWLIST.has(origin)) return callback(null, true);

    // Not allowed: pass an error. `cors` will NOT set ACAO, so the browser blocks
    // the read. We do NOT throw a 500 here to avoid leaking that CORS was the cause;
    // returning false is gentler, but an error lets you log the rejected origin.
    return callback(new Error(`CORS: origin ${origin} not allowed`));
  },
  credentials: true,   // emit Access-Control-Allow-Credentials: true (needs exact origin, not *)
  maxAge: 600,         // cache preflight 10 min to cut OPTIONS round-trips
};

// Mount BEFORE auth and routes so preflight OPTIONS is answered without authentication.
app.use(cors(corsOptions));

app.get('/api/products', (req, res) => {
  res.json([{ id: 1, name: 'Widget' }]);
});

app.listen(4000);
```

Line-by-line rationale:

- **`ALLOWLIST` as a `Set` of exact strings.** No regex-by-default, no substring matching — a substring check (`origin.includes('example.com')`) is a classic vulnerability because `https://example.com.evil.net` and `https://evilexample.com` both pass. Exact match is the safe baseline.
- **`origin` callback vs a static string.** A static `origin: 'https://app.example.com'` only supports one front-end. The callback supports many, and reflects the *matched* origin so credentials work. `cors` sets `Vary: Origin` automatically whenever the reflected value depends on the request — do not remove it.
- **`if (!origin) return callback(null, true)`.** Requests with no `Origin` are same-origin or non-browser. Rejecting them breaks curl-based health checks, server-side rendering fetches, and monitoring. They aren't a CORS concern (no browser is enforcing anything). Note this does *not* weaken browser security — a real cross-origin browser request always carries `Origin`.
- **`credentials: true`.** Emits `Access-Control-Allow-Credentials: true`. This *requires* the exact-origin reflection above; combined with `*` the browser would reject the response. See [Access-Control-Allow-Credentials](./Access-Control-Allow-Credentials.md).
- **`maxAge: 600`.** Caches the preflight so repeated calls skip the `OPTIONS`. See [Access-Control-Allow-Methods](./Access-Control-Allow-Methods.md) for `Max-Age` mechanics.
- **Mount order.** If your JWT/session middleware runs first, it rejects the credential-less preflight `OPTIONS` with 401 and CORS breaks. `cors` must be earlier.

If you truly want a **public, anonymous, cacheable** API (no cookies, no auth-by-cookie), use the simpler and more cache-friendly:

```js
app.use(cors({ origin: '*' }));   // Access-Control-Allow-Origin: *, no Vary needed, no credentials
```

## Node.js Example

Without Express, using the raw `http` module — this shows exactly what the middleware does under the hood:

```js
const http = require('http');

const ALLOWLIST = new Set(['https://app.example.com', 'https://admin.example.com']);

const server = http.createServer((req, res) => {
  const origin = req.headers.origin;

  if (origin && ALLOWLIST.has(origin)) {
    // Reflect the exact origin — never `*` when credentials are involved.
    res.setHeader('Access-Control-Allow-Origin', origin);
    res.setHeader('Access-Control-Allow-Credentials', 'true');
    // MANDATORY: the response body/headers depend on Origin, so caches must key on it.
    res.setHeader('Vary', 'Origin');
  }

  // Answer the preflight ourselves. The browser sends OPTIONS before non-simple requests.
  if (req.method === 'OPTIONS') {
    res.setHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE');
    res.setHeader('Access-Control-Allow-Headers', 'Authorization, Content-Type');
    res.setHeader('Access-Control-Max-Age', '600');
    res.writeHead(204);   // 204 No Content — preflights have no body
    return res.end();
  }

  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify({ ok: true }));
});

server.listen(4000);
```

The key manual chore the middleware hides: you must set `Vary: Origin` yourself, and you must handle `OPTIONS` explicitly, or preflighted requests silently fail.

## React Example

React never sets ACAO — it's a *response* header your server controls. React's only role is being *served from an origin* and issuing `fetch`/`axios` calls to another origin. The failure a React dev sees is in the console, and the fix is on the server. What React controls is the request side:

```jsx
import { useEffect, useState } from 'react';

function Products() {
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);

  useEffect(() => {
    // This runs on https://app.example.com and calls https://api.example.com — cross-origin.
    // The browser adds Origin automatically; you cannot and need not set it.
    fetch('https://api.example.com/api/products', {
      credentials: 'include',   // send cookies cross-origin → server MUST reflect exact origin, not *
    })
      .then((res) => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      })
      .then(setData)
      .catch((e) => {
        // A CORS block surfaces here as an opaque "TypeError: Failed to fetch".
        // The real cause (missing/mismatched ACAO) is only visible in the Network tab.
        setError(e.message);
      });
  }, []);

  if (error) return <p>Failed: {error}</p>;
  return <pre>{JSON.stringify(data, null, 2)}</pre>;
}
```

The teaching point for React engineers: **a CORS failure is indistinguishable in JS from a network failure** — you get a generic `TypeError: Failed to fetch` with no detail, because exposing the response to your error handler would itself violate the policy. Always diagnose in the Network tab, not the JS console.

## Browser Lifecycle

1. Script calls `fetch` to a cross-origin URL; browser tags the request with the [`Origin`](../03-Request-Headers/Origin.md) of the calling document.
2. If the request is non-simple, the browser first runs a preflight and checks ACAO on the `OPTIONS` response; a mismatch aborts before the real request.
3. The browser sends the actual request with `Origin`.
4. Response arrives; the browser reads `Access-Control-Allow-Origin`.
5. **Match test:** value is `*` (and not credentialed) OR value string-equals the request `Origin`.
6. On match, the response is exposed to the script (subject to `Access-Control-Expose-Headers` for reading custom headers). On mismatch/absence, the browser discards the response and rejects the promise — the server-side side effects already occurred.

## Production Use Cases

- **SPA + API split** (`app.example.com` → `api.example.com`): allowlist reflection with credentials.
- **Public read-only API / open data**: static `Access-Control-Allow-Origin: *`, no credentials, highly cacheable.
- **Multi-tenant / white-label**: dynamically reflect any origin matching a tenant-owned domain checked against a database, with `Vary: Origin`.
- **Static assets served cross-origin** (fonts, WebGL textures, canvas images): fonts and `crossorigin`-loaded assets require ACAO or the browser refuses to use them; `*` is typical for a public asset CDN.
- **Wildcard subdomains**: an org with `*.example.com` front-ends reflects any subdomain that validates against a pattern (see Common Mistakes for the safe way).

## Common Mistakes

- **`Access-Control-Allow-Origin: *` with credentials.** Illegal; the browser rejects it. Reflect the exact origin instead. See [Access-Control-Allow-Credentials](./Access-Control-Allow-Credentials.md).
- **Sending a *list* of origins.** `Access-Control-Allow-Origin: https://a.com, https://b.com` is invalid — the header takes exactly one origin or `*`. Reflect one origin per request instead.
- **Substring / naive matching.** `origin.endsWith('example.com')` matches `notexample.com`; `origin.includes('example.com')` matches `example.com.evil.net`. Both are exploitable. Use exact set membership or a strictly anchored regex.
- **Reflecting `Origin` unconditionally.** `res.setHeader('Access-Control-Allow-Origin', req.headers.origin)` with no allowlist plus `Allow-Credentials: true` is a total CORS bypass — you've made *every* website able to read authenticated responses. This is a real, common vulnerability.
- **Forgetting [`Vary: Origin`](../06-Caching-Headers/Vary.md).** Reflected ACAO without `Vary: Origin` gets cache-poisoned across origins.
- **Missing ACAO on error responses.** A 4xx/5xx that skips CORS headers turns your real error into an opaque CORS error in the browser; add CORS headers to error paths too.
- **Trailing slash / scheme mismatch.** `https://app.example.com/` will never match `https://app.example.com`.

## Security Considerations

The dominant risk is **origin-reflection without validation**. If you reflect whatever `Origin` arrives and also allow credentials, any malicious site can, using a logged-in victim's cookies, read authenticated responses from your API — a full same-origin bypass. Rules:

1. Reflect only origins on a vetted allowlist. Never reflect blindly.
2. Match origins exactly (or with a tightly anchored, tested regex). Guard against `null` origin (sandboxed iframes, `file://`, some redirects send `Origin: null`) — do **not** allowlist `null`, as multiple hostile contexts share it.
3. Remember CORS is browser-only enforcement: ACAO does not protect your endpoint from `curl`/server clients. It only governs which *websites' scripts* can read responses in a browser. Sensitive endpoints still need real authentication and authorization ([see the Overview](./CORS-Overview.md)).
4. For subdomain wildcarding, validate the origin against a regex like `^https:\/\/([a-z0-9-]+\.)?example\.com$` and reflect the matched value — do **not** try `Access-Control-Allow-Origin: *.example.com` (invalid syntax; `*` cannot be partial).

## Performance Considerations

- **`*` is the most cache-friendly value** because the response is identical for all origins — no `Vary`, one cache entry. Prefer it for public, anonymous data.
- **Reflected ACAO fragments the cache** by origin (`Vary: Origin` means one entry per origin), reducing hit rate. Acceptable and necessary for private/credentialed APIs, but don't reflect when `*` would do.
- ACAO itself adds negligible bytes; the real cost is the *preflight* it participates in — mitigate with [`Access-Control-Max-Age`](./Access-Control-Allow-Methods.md).

## Reverse Proxy Considerations

Nginx example that reflects an allowlisted origin and always includes it (even on errors):

```nginx
map $http_origin $cors_origin {
    default "";
    "https://app.example.com"   $http_origin;
    "https://admin.example.com" $http_origin;
}

server {
    location /api/ {
        # `always` ensures the header is added on 4xx/5xx too, not just 2xx —
        # without it, errors surface to the browser as opaque CORS failures.
        add_header Access-Control-Allow-Origin $cors_origin always;
        add_header Access-Control-Allow-Credentials "true" always;
        add_header Vary Origin always;

        if ($request_method = OPTIONS) {
            add_header Access-Control-Allow-Methods "GET, POST, PUT, DELETE" always;
            add_header Access-Control-Allow-Headers "Authorization, Content-Type" always;
            add_header Access-Control-Max-Age 600 always;
            return 204;
        }

        proxy_pass http://app_upstream;
    }
}
```

Gotchas: Nginx `add_header` is dropped if any `add_header` appears in a more specific block (they don't merge — you must repeat them); and Nginx `add_header` on 4xx/5xx requires the `always` flag. Decide whether the proxy *or* the app owns CORS — doing both often produces duplicate ACAO headers, which browsers reject.

## CDN Considerations

- Configure the CDN to include the `Origin` request header in the **cache key** whenever ACAO is reflected, or set `Vary: Origin` and ensure the CDN honors `Vary` (many CDNs ignore `Vary` on non-`Accept-Encoding` values unless told otherwise — Cloudflare, for instance, historically ignores arbitrary `Vary`). If the CDN can't vary on `Origin`, either serve `*` (public data) or bypass cache for credentialed API routes.
- Some CDNs let you rewrite/inject ACAO at the edge; ensure the edge logic uses the same allowlist as the origin to avoid drift.

## Cloud Deployment Considerations

- **AWS API Gateway**: CORS is configured per-resource; for REST APIs you must enable CORS *and* often add a mock `OPTIONS` integration. API Gateway can only emit a single static origin unless you use a Lambda/proxy integration to reflect dynamically. `Access-Control-Allow-Origin: '*'` is the console default — override for credentialed APIs.
- **AWS ALB**: doesn't add CORS; your app or a Lambda must. Watch for the ALB not forwarding headers you expect.
- **Cloudflare Workers / Lambda@Edge**: ideal place to centralize allowlist reflection + `Vary` at the edge.
- **Google Cloud / Azure API gateways**: similar — CORS is a gateway feature; verify it varies on `Origin` and covers error responses.

## Debugging

- **Chrome DevTools → Network tab**: click the request; the *General* section shows the status, and *Response Headers* shows whether ACAO is present and its value. The Console shows the specific CORS reason ("No 'Access-Control-Allow-Origin' header is present" vs "value `*` not allowed with credentials"). The preflight appears as a separate `OPTIONS` row — inspect it too.
- **curl**: `curl -i -H "Origin: https://app.example.com" https://api.example.com/api/products` — check for `Access-Control-Allow-Origin` in the output. For preflight: `curl -i -X OPTIONS -H "Origin: https://app.example.com" -H "Access-Control-Request-Method: PUT" https://api.example.com/resource`.
- **Postman / Bruno**: useful to confirm the server *emits* the right headers, but remember they don't enforce CORS, so a passing Postman call proves nothing about browser behavior — only that the header exists in the response.
- **Node/Express logging**: log `req.headers.origin` and the ACAO you set. Log rejected origins in the allowlist callback to catch legitimate front-ends you forgot to add.

## Best Practices

- ✅ Use an **exact-match allowlist** with dynamic reflection for private/credentialed APIs; use `*` only for public, anonymous, cacheable data.
- ✅ Always pair reflected ACAO with [`Vary: Origin`](../06-Caching-Headers/Vary.md).
- ✅ Never combine `*` with credentials; the browser will reject it.
- ✅ Emit CORS headers on **error responses** too.
- ✅ Never reflect `Origin` without validation; never use substring matching; never allowlist `null`.
- ✅ Prefer the `cors` middleware over hand-rolled headers; keep one owner (app *or* proxy).
- ✅ Set `Access-Control-Max-Age` to amortize preflights.

## Related Headers

- [`Origin`](../03-Request-Headers/Origin.md) — the request header ACAO is matched against.
- [`Access-Control-Allow-Credentials`](./Access-Control-Allow-Credentials.md) — forces exact-origin ACAO (no `*`).
- [`Access-Control-Allow-Methods`](./Access-Control-Allow-Methods.md) / [`Access-Control-Allow-Headers`](./Access-Control-Allow-Headers.md) — the other preflight grants.
- [`Vary`](../06-Caching-Headers/Vary.md) — mandatory alongside reflected ACAO.
- `Access-Control-Expose-Headers` — controls which response headers the script may read *after* ACAO permits the read.

## Decision Tree

```
Is the endpoint public, anonymous data (no cookies/auth-by-cookie)?
├── YES → Access-Control-Allow-Origin: *   (no credentials, no Vary needed, cache-friendly)
└── NO (private / uses credentials)
     └── Do you have a fixed, known set of front-end origins?
          ├── YES → reflect from an exact-match allowlist + Vary: Origin + Allow-Credentials: true
          └── NO (dynamic tenant domains)
               └── Validate Origin against tenant records / anchored regex,
                   reflect the matched value + Vary: Origin.
                   Never reflect unvalidated. Never allowlist `null`.
```

## Mental Model

**ACAO is a one-name guest pass the server prints for the browser's bouncer.** The pass names exactly one origin — or says "open house" (`*`). The browser-bouncer already let the request into the kitchen (the server ran the handler), but on the way *out* it checks whether the script's origin matches the name on the pass before letting it walk off with the food (the response). A pass that says "open house" is fine for a public buffet, but you must never print "open house" *and* hand over the guest's wallet (`credentials`) — so for anything valuable, print the guest's exact name, and tell the coat-check (the cache) that this plate was made for that specific guest (`Vary: Origin`).
