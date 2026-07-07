# Allow

## Quick Summary

`Allow` is a **response** header that lists the **HTTP methods a resource supports** — e.g. `Allow: GET, POST, HEAD, OPTIONS`. Its two canonical uses are: on a **`405 Method Not Allowed`** response, where it is **mandatory** (it tells the client *which* methods it should have used instead of the one it tried), and on an **`OPTIONS`** response, where it advertises the resource's method capabilities. It is a *resource-level* capability declaration — "here is what you can do to this URL" — and it's distinct from the CORS [`Access-Control-Allow-Methods`](../07-CORS/Access-Control-Allow-Methods.md) header, which governs *cross-origin* method permission during preflight. `Allow` is about the resource's inherent API surface; `Access-Control-Allow-Methods` is about the browser's cross-origin security gate. Getting `Allow` right makes your API self-describing and your `405`s actionable; omitting it on a `405` is a spec violation that leaves clients guessing.

## What problem does this header solve?

When a client sends a request with a method the resource doesn't support — a `DELETE` to a read-only endpoint, a `POST` to a static file — the server rejects it with `405 Method Not Allowed`. But a bare `405` is unhelpful: the client knows *this* method failed but not *which* methods would work. Should it try `PUT`? `PATCH`? Give up? Without guidance, clients (and the humans debugging them) are left guessing, and API discovery becomes trial-and-error.

`Allow` solves this by making the rejection **actionable**: alongside the `405`, the server lists exactly the methods the resource accepts, so the client (or developer) immediately knows the valid options. It also solves **capability discovery** on `OPTIONS`: a client can ask "what can I do with this URL?" and get a definitive method list without probing each verb. This makes APIs self-documenting at the protocol level — tooling, generated clients, and humans can learn a resource's supported operations directly from HTTP.

## Why was it introduced?

`Allow` has been part of HTTP since **HTTP/1.0 (RFC 1945, 1996)** and HTTP/1.1, specified today in **RFC 9110 §10.2.1 (2022)**. It was introduced as the natural companion to method negotiation: HTTP defines a set of methods (`GET`, `POST`, `PUT`, `DELETE`, etc.), resources support different subsets, and both the `405 Method Not Allowed` status and the `OPTIONS` method need a way to *communicate that subset*. The spec makes `Allow` **required on `405`** precisely so the error is self-correcting — a client that sent the wrong method learns the right ones from the same response. Its pairing with `OPTIONS` supports capability discovery, which became more important with REST APIs and automated tooling that inspect resources programmatically.

## How does it work?

The server sets `Allow` to a comma-separated list of the methods valid for the target resource, on `405` responses (mandatory) and typically on `OPTIONS` responses (advertisement).

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    C->>S: DELETE /articles/42
    Note over S: This resource only supports GET/HEAD/OPTIONS
    S-->>C: 405 Method Not Allowed<br/>Allow: GET, HEAD, OPTIONS
    Note over C: Learn valid methods; retry with GET
    C->>S: OPTIONS /articles/42
    S-->>C: 204 No Content<br/>Allow: GET, HEAD, OPTIONS
```

- **Browser behavior:** Browsers don't act on `Allow` automatically for normal navigation; it's read by developers and by tooling/SDKs. It's a *response* header, so JS can read it (subject to CORS-safelisting rules for cross-origin — `Allow` is not on the safelist, so cross-origin JS needs [`Access-Control-Expose-Headers`](../07-CORS/Access-Control-Expose-Headers.md) to read it).
- **Server behavior:** The origin computes the method set for the resource and emits `Allow` on `405` (required) and `OPTIONS` (recommended). Frameworks often generate it automatically from route definitions.
- **Proxy behavior:** Forwards `Allow` untouched.
- **CDN behavior:** Passes it through; CDNs generally don't synthesize it.
- **Reverse proxy behavior:** Nginx returns `405` for disallowed methods in some configs and can set `Allow`; the value should reflect the *application's* method set, which is why it's usually set by the app.

## HTTP Request Example

A request using an unsupported method (which will get a `405 + Allow`):

```http
DELETE /articles/42 HTTP/1.1
Host: blog.example.com
```

A capability probe:

```http
OPTIONS /articles/42 HTTP/1.1
Host: blog.example.com
```

## HTTP Response Example

Mandatory `Allow` on a `405`:

```http
HTTP/1.1 405 Method Not Allowed
Allow: GET, HEAD, OPTIONS
Content-Type: application/json

{"error":"method_not_allowed","supported":["GET","HEAD","OPTIONS"]}
```

Capability advertisement on `OPTIONS`:

```http
HTTP/1.1 204 No Content
Allow: GET, POST, PUT, DELETE, HEAD, OPTIONS
```

An empty `Allow` (rare, but legal) means the resource supports *no* methods currently:

```http
HTTP/1.1 405 Method Not Allowed
Allow:
```

## Express.js Example

Express can generate `405 + Allow` automatically if you structure routes with `app.route`/routers, or you set it explicitly:

```js
const express = require('express');
const app = express();

// 1) Explicit method handling with a correct 405 + Allow for the mismatched method.
const router = express.Router();
router.route('/articles/:id')
  .get((req, res) => res.json(getArticle(req.params.id)))
  .put((req, res) => res.json(updateArticle(req.params.id, req.body)))
  .all((req, res) => {
    // Reached only if the method wasn't GET/PUT above → advertise the real set.
    res.set('Allow', 'GET, PUT, OPTIONS');   // MANDATORY on 405.
    res.status(405).json({ error: 'method_not_allowed', supported: ['GET', 'PUT', 'OPTIONS'] });
  });
app.use(router);

// 2) Answer OPTIONS with the capability list for discovery/tooling.
app.options('/articles/:id', (req, res) => {
  res.set('Allow', 'GET, PUT, OPTIONS');
  res.status(204).end();   // OPTIONS with Allow, no body.
});

// 3) A read-only resource: reject writes with a precise Allow.
app.all('/config', (req, res, next) => {
  if (req.method === 'GET' || req.method === 'HEAD') return next();
  res.set('Allow', 'GET, HEAD').status(405).end();
});
app.get('/config', (req, res) => res.json(loadConfig()));

app.listen(3000);
```

Why each piece matters: the `.all()` catch-all in route 1 handles any method not explicitly defined and sets `Allow` — without it, Express would return a generic `404`/`405` (depending on setup) *without* the `Allow` header, violating the spec and leaving clients uninformed. The `Allow` value must **match the methods actually implemented** for that route — listing `DELETE` when there's no delete handler misleads clients. Answering `OPTIONS` (route 2) makes the resource self-describing for API explorers and generated clients. Note: this `Allow` is about the *resource's* methods, entirely separate from CORS preflight — a cross-origin `DELETE` also needs [`Access-Control-Allow-Methods`](../07-CORS/Access-Control-Allow-Methods.md) on the preflight, which is a different concern.

## Node.js Example

Raw `http`:

```js
const http = require('http');

const ROUTES = {
  '/articles': ['GET', 'POST', 'OPTIONS'],
  '/config': ['GET', 'HEAD', 'OPTIONS'],
};

http.createServer((req, res) => {
  const allowed = ROUTES[req.url];
  if (!allowed) { res.statusCode = 404; return res.end(); }

  if (req.method === 'OPTIONS') {
    res.setHeader('Allow', allowed.join(', '));
    res.statusCode = 204;
    return res.end();
  }

  if (!allowed.includes(req.method)) {
    // 405 MUST carry Allow listing the supported methods.
    res.setHeader('Allow', allowed.join(', '));
    res.statusCode = 405;
    return res.end();
  }

  // ... handle the (allowed) method
  res.statusCode = 200;
  res.end('ok');
}).listen(3000);
```

The rule made explicit: whenever you return `405`, set `Allow` to the resource's method set; answer `OPTIONS` with the same list.

## React Example

React rarely reads `Allow` directly, but it surfaces in API tooling and error handling:

1. **Actionable 405 handling.** When a fetch returns `405`, reading `Allow` tells your code (or an error message) which methods are valid — useful for building generic API clients or dev tooling:

```jsx
async function callApi(url, method, body) {
  const res = await fetch(url, { method, body: body && JSON.stringify(body) });
  if (res.status === 405) {
    // Cross-origin: needs Access-Control-Expose-Headers: Allow to read this.
    const allowed = res.headers.get('allow');
    throw new Error(`Method ${method} not allowed. Supported: ${allowed}`);
  }
  return res.json();
}
```

2. **Capability-driven UI.** An admin tool can `OPTIONS` a resource and read `Allow` to decide which action buttons (Edit/Delete) to show — hiding actions the API doesn't support.

3. **Cross-origin caveat.** To read `Allow` from a cross-origin response in JS, the server must send [`Access-Control-Expose-Headers: Allow`](../07-CORS/Access-Control-Expose-Headers.md) (it's not CORS-safelisted).

## Browser Lifecycle

1. A request uses an unsupported method → server responds `405` with `Allow`.
2. The browser surfaces the response to `fetch`/XHR; it does **not** auto-retry with an allowed method — that's your code's job.
3. For `OPTIONS` (non-preflight, explicit), the browser returns the response with `Allow` to your code.
4. Cross-origin readability of `Allow` requires [CORS exposure](../07-CORS/Access-Control-Expose-Headers.md).
5. Note: a CORS **preflight** `OPTIONS` is handled by the browser internally and uses [`Access-Control-Allow-Methods`](../07-CORS/Access-Control-Allow-Methods.md), *not* `Allow` — the two are unrelated in that flow.

## Production Use Cases

- **Spec-compliant `405` responses:** always include `Allow` so clients/developers know the valid methods.
- **API discovery via `OPTIONS`:** let tooling, generated SDKs, and API explorers learn a resource's methods.
- **Capability-driven UIs:** show/hide actions based on a resource's supported methods.
- **Read-only endpoint enforcement:** reject writes with `405 + Allow: GET, HEAD`.
- **HATEOAS / self-describing APIs:** advertise available operations at the protocol level.
- **Gateway/method routing:** an API gateway rejecting unsupported methods with an accurate `Allow`.

## Common Mistakes

- **Omitting `Allow` on `405`.** It's **required**; a `405` without it is non-compliant and unhelpful.
- **Confusing it with [`Access-Control-Allow-Methods`](../07-CORS/Access-Control-Allow-Methods.md).** `Allow` = resource's methods (any client); ACAM = cross-origin preflight permission (browsers). Setting one doesn't satisfy the other.
- **Listing methods you don't implement.** Misleads clients; keep `Allow` in sync with actual handlers.
- **Returning `404` instead of `405`** for a known resource with a wrong method — hides the fact that the URL exists and skips `Allow`.
- **Not answering `OPTIONS`.** Leaves the resource non-discoverable (and, for CORS, can break preflight — though that uses ACAM).
- **Forgetting cross-origin exposure.** JS can't read `Allow` cross-origin without [`Access-Control-Expose-Headers`](../07-CORS/Access-Control-Expose-Headers.md).
- **Inconsistent casing/spacing.** Use standard uppercase method names, comma-separated.

## Security Considerations

- **Method surface disclosure.** `Allow` reveals which operations a resource supports, which can inform an attacker (e.g. discovering that `DELETE` or `PUT` is available). This is usually acceptable (it's part of a usable API), but for sensitive endpoints, don't advertise methods you intend to keep obscure — and never rely on *hiding* a method for security (obscurity ≠ authorization).
- **Not authorization.** `Allow` says a method is *supported*, not that the caller is *permitted* — always enforce authz per request; a `405` vs `403` distinction shouldn't leak more than intended.
- **`TRACE`/`TRACK` and dangerous methods.** Don't advertise (or support) methods like `TRACE` that enable Cross-Site Tracing; disable them and exclude from `Allow`.
- **Consistency with actual behavior.** If `Allow` lists a method the server mishandles, it can invite unexpected requests; keep it truthful and tested.

## Performance Considerations

- **Negligible cost;** a small header on `405`/`OPTIONS` responses only.
- **Reduces trial-and-error round-trips:** clients learn valid methods immediately instead of probing each verb.
- **`OPTIONS` responses should be cheap** (no DB work, small/empty body) — they're pure discovery overhead.
- **No caching interaction of its own,** though `OPTIONS`/`405` responses are generally not cached.

## Reverse Proxy Considerations

Nginx can return `405` for disallowed methods and set `Allow`, but the method set usually belongs to the app:

```nginx
server {
  location /config {
    limit_except GET HEAD {
      deny all;                 # returns 403 by default; see note below.
    }
    proxy_pass http://app_upstream;
  }

  # To return a proper 405 + Allow at the edge:
  location /readonly {
    if ($request_method !~ ^(GET|HEAD|OPTIONS)$) {
      add_header Allow "GET, HEAD, OPTIONS" always;
      return 405;
    }
    proxy_pass http://app_upstream;
  }
}
```

Key points: `limit_except` returns `403` (not `405`) by default, which is *not* method-semantic — for a spec-correct `405 + Allow`, use an explicit `if` + `return 405` with `add_header Allow` (as in `/readonly`), or let the application own it. Generally, prefer setting `Allow` in the app where the true method set lives.

## CDN Considerations

- **Pass-through:** CDNs forward `Allow` from the origin; they don't usually generate it.
- **Edge method restrictions:** if a CDN/WAF blocks certain methods, ensure it returns `405 + Allow` (not a generic block) where you want method semantics.
- **CORS distinction:** remember the CDN's CORS features use [`Access-Control-Allow-Methods`](../07-CORS/Access-Control-Allow-Methods.md) for preflight, separate from `Allow`.
- **Expose for cross-origin JS:** ensure [`Access-Control-Expose-Headers: Allow`](../07-CORS/Access-Control-Expose-Headers.md) isn't stripped if browsers need to read it.

## Cloud Deployment Considerations

- **API Gateways (AWS API Gateway, Apigee, Kong):** return `405` for undefined methods; configure them to include an accurate `Allow` (some do automatically). Define `OPTIONS` handlers for discovery/CORS.
- **Load balancers:** pass `Allow` through; generally don't generate it.
- **Serverless:** your function/router must set `Allow` on `405` and answer `OPTIONS` — frameworks (Express, Fastify) can do this from route definitions.
- **Managed platforms (Vercel/Netlify):** method handling is in your function/middleware; ensure `405 + Allow` and `OPTIONS` are handled.

## Debugging

- **curl (405):** `curl -i -X DELETE https://host/articles/42` → expect `405` and an `Allow` header listing valid methods.
- **curl (OPTIONS):** `curl -i -X OPTIONS https://host/articles/42` → inspect `Allow`.
- **Chrome DevTools → Network:** on a `405`, check Response Headers for `Allow`; the Console shows the error.
- **Postman / Bruno:** assert `res.status === 405` and that `Allow` contains the expected methods; send `OPTIONS` to read capabilities.
- **Node.js/Express:** log `res.getHeader('allow')` alongside the status to confirm it's set on every `405`.
- **Cross-origin:** verify `Access-Control-Expose-Headers: Allow` if a browser app needs to read it.

## Best Practices

- [ ] Always include `Allow` on **`405`** responses (it's mandatory) with the resource's real method set.
- [ ] Answer **`OPTIONS`** with `Allow` for discovery and self-describing APIs.
- [ ] Keep `Allow` **in sync** with the methods you actually implement.
- [ ] Return `405` (not `404`) when a known resource is hit with an unsupported method.
- [ ] Don't confuse `Allow` with [`Access-Control-Allow-Methods`](../07-CORS/Access-Control-Allow-Methods.md) — set both when doing cross-origin method-restricted APIs.
- [ ] Exclude dangerous methods (`TRACE`) and disable them server-side.
- [ ] Expose `Allow` via [`Access-Control-Expose-Headers`](../07-CORS/Access-Control-Expose-Headers.md) if browser JS must read it cross-origin.
- [ ] Enforce authorization per request — `Allow` is capability, not permission.

## Related Headers

- [Access-Control-Allow-Methods](../07-CORS/Access-Control-Allow-Methods.md) — the CORS preflight counterpart; governs cross-origin method permission (different purpose).
- [Access-Control-Expose-Headers](../07-CORS/Access-Control-Expose-Headers.md) — needed for cross-origin JS to read `Allow`.
- [Accept](../03-Request-Headers/Accept.md) — content negotiation counterpart (media types, not methods).
- [Retry-After](./Retry-After.md) — another actionable-error header (for `429`/`503`).
- [Content-Type](./Content-Type.md) — often paired in an error body describing supported methods.
- [Custom and X- Headers](../02-Core-Concepts/Custom-and-X-Headers.md) — for context on method/capability conventions.

## Decision Tree

```mermaid
flowchart TD
    A[Request arrives] --> B{Is the URL a known resource?}
    B -- No --> C[404 Not Found]
    B -- Yes --> D{Method supported by this resource?}
    D -- No --> E[405 Method Not Allowed<br/>+ Allow: supported methods (REQUIRED)]
    D -- Yes --> F[Handle the method]
    A --> G{OPTIONS request?}
    G -- Yes --> H[204 + Allow: capability list]
    E --> I{Cross-origin browser needs to read Allow?}
    I -- Yes --> J[Add Access-Control-Expose-Headers: Allow]
```

## Mental Model

Think of `Allow` as the **"We're open for: dine-in, takeout, delivery" sign on a restaurant door** — it tells you, up front, exactly which ways you can interact with this establishment. If you walk in demanding *catering* and they don't offer it, a good host doesn't just say "no" and walk away (a bare `405`); they say "we don't cater, but we do dine-in, takeout, and delivery" (`405` **+** `Allow`), so you immediately know your real options. And if you're just scouting, you can ask "what do you offer?" (`OPTIONS`) and get the same list without ordering one of everything to find out. The important distinction: this sign describes *what the restaurant fundamentally does* — it's completely separate from the *bouncer at a members-only cross-town club* deciding whether an outside visitor may enter ([`Access-Control-Allow-Methods`](../07-CORS/Access-Control-Allow-Methods.md), the CORS gate). One is the menu of operations; the other is a security checkpoint for visitors from other origins.
