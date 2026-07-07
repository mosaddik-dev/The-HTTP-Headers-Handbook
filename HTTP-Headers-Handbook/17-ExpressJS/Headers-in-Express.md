# Headers in Express

> Express is the layer where the abstract HTTP header model you've read about in the rest of this handbook becomes concrete method calls, middleware ordering, and Node's `http.ServerResponse` state machine. Nearly every header bug in a Node stack is really one of three things: reading a header from the wrong place (raw socket vs. proxy-rewritten), writing a header after the response has already committed, or middleware running in an order that clobbers what an earlier layer set. This chapter is about the mechanics underneath `req` and `res` — not the headers themselves (those have their own chapters), but *how Express lets you touch them* and where the sharp edges are.

Express is a thin, opinionated wrapper over Node's core `http` module. `req` is an augmented [`http.IncomingMessage`](https://nodejs.org/api/http.html#class-httpincomingmessage); `res` is an augmented [`http.ServerResponse`](https://nodejs.org/api/http.html#class-httpserverresponse). Everything Express adds — `req.get()`, `res.set()`, `req.hostname`, `res.cookie()` — is sugar over `req.headers`, `res.setHeader()`, and Node's write machinery. Understanding that inheritance is the key to understanding why the sharp edges are where they are: the moment you drop below Express's helpers to raw Node, the same rules apply, just less politely.

## Reading request headers

### `req.headers` — the parsed object

Node parses the incoming header block into `req.headers`, a plain object with **lowercased keys**:

```js
app.get('/', (req, res) => {
  console.log(req.headers);
  // {
  //   host: 'api.example.com',
  //   'user-agent': 'curl/8.4.0',
  //   accept: '*/*',
  //   'content-type': 'application/json',
  //   'x-request-id': 'abc-123'
  // }
});
```

Header names in HTTP are **case-insensitive** (RFC 9110 §5.1), so Node normalizes every field name to lowercase on parse. This matters: `req.headers.Authorization` is `undefined`; `req.headers.authorization` is the value. Never rely on the client's casing.

**Duplicate headers** are handled by Node with header-specific rules, not a single universal one:

- Most headers: the first occurrence wins, duplicates are **discarded** (e.g. `Content-Type`, `User-Agent`).
- `Set-Cookie`: **always an array**, because multiple cookies are the norm and each is a distinct value.
- A defined set of comma-safe headers (`Cookie`, and per Node's list) are **joined with `, `** into one string.

```js
// Two "X-Forwarded-For: a" and "X-Forwarded-For: b" arrive:
req.headers['x-forwarded-for']; // 'a, b'  (comma-joined)
req.headers['set-cookie'];      // ['a=1', 'b=2']  (array)
```

If you need to reason precisely about which merge rule applies, don't — read `req.rawHeaders` instead (below). Relying on the merge behavior for a security-sensitive header like `X-Forwarded-For` is how request-smuggling and IP-spoofing bugs get in; see [X-Forwarded-For](../14-Proxies/X-Forwarded-For.md).

### `req.get(name)` / `req.header(name)` — the accessor

`req.get()` is the idiomatic way to read one header. `req.header()` is an alias — identical behavior, and the name `header` is retained only for historical reasons; prefer `req.get()` for consistency with `res.set()`/`res.get()`.

```js
req.get('Content-Type');   // 'application/json'
req.get('content-type');   // 'application/json' — case-insensitive lookup
req.get('Referer');        // works
req.get('Referrer');       // ALSO works — Express special-cases the famous misspelling
```

Two reasons to prefer `req.get()` over reaching into `req.headers` directly:

1. **Case-insensitive lookup.** You can write `req.get('Content-Type')` in natural casing; internally it lowercases before looking up. Reaching into `req.headers['Content-Type']` (capital C) silently returns `undefined`.
2. **`Referer`/`Referrer` aliasing.** The HTTP header is famously misspelled `Referer`. Express lets you ask for either spelling. `req.headers` does not.

What breaks if you skip it: reading `req.headers['Authorization']` (capitalized) returns `undefined`, your auth check treats every request as unauthenticated, and you burn an hour before noticing the capital A.

### `req.rawHeaders` — the unparsed truth

`req.rawHeaders` is a **flat array** of the header block exactly as received, preserving original casing and every duplicate, in `[key, value, key, value, ...]` order:

```js
req.rawHeaders;
// [ 'Host', 'api.example.com',
//   'X-Forwarded-For', 'a',
//   'X-Forwarded-For', 'b',
//   'Content-Type', 'application/json' ]
```

You need this in exactly two situations: (1) debugging a proxy or client that sends surprising casing/duplicates, and (2) security-sensitive parsing where the merge behavior of `req.headers` would hide an attack (e.g. detecting a smuggled duplicate `Content-Length` or `Transfer-Encoding`). For everything else, `req.get()` is correct and `rawHeaders` is noise.

## Derived request properties: hostname, ip, protocol, secure

Express exposes several *computed* properties that are not headers themselves but are **derived from headers**, and their derivation changes based on your `trust proxy` setting. This is where a huge fraction of production Express bugs live.

| Property | Derived from | Notes |
|---|---|---|
| `req.hostname` | `Host` header (or `X-Forwarded-Host` if trusting proxy) | Host name only, port stripped |
| `req.ip` | Socket remote address (or leftmost trusted `X-Forwarded-For` entry) | The client's IP *as Express understands it* |
| `req.ips` | `X-Forwarded-For` chain | Array, only populated when trusting proxy |
| `req.protocol` | `'https'` if TLS-terminated at Node, else `'http'` (or `X-Forwarded-Proto` if trusting proxy) | |
| `req.secure` | `req.protocol === 'https'` | Convenience boolean |
| `req.subdomains` | `req.hostname` split on `.` | Offset controlled by `subdomain offset` setting |

The critical insight: **when Node is behind a load balancer or reverse proxy that terminates TLS**, the socket that reaches Node is a plain-HTTP connection from the proxy, on an internal IP. Left alone, `req.protocol` reports `'http'`, `req.secure` is `false`, and `req.ip` is the proxy's address — all wrong from the client's perspective. The proxy communicates the real values via forwarding headers (`X-Forwarded-Proto`, `X-Forwarded-For`, `X-Forwarded-Host`), and Express only reads them **if you tell it to trust the proxy.**

### `app.set('trust proxy', ...)` — and its security caveat

`trust proxy` controls whether Express believes the `X-Forwarded-*` headers, and if so, *how many hops* to trust.

```js
// Trust the first proxy hop (typical single load balancer):
app.set('trust proxy', 1);

// Trust a specific proxy subnet / named ranges:
app.set('trust proxy', 'loopback, 10.0.0.0/8');

// Trust everything (DANGEROUS — see below):
app.set('trust proxy', true);

// Custom predicate:
app.set('trust proxy', (ip, hopIndex) => ip === '10.0.0.1');
```

With `trust proxy` enabled, the derivations change:

- `req.protocol` reads `X-Forwarded-Proto` (`https`) instead of the socket.
- `req.ip` becomes the **rightmost-untrusted** entry in the `X-Forwarded-For` chain — i.e. Express walks the chain from the right, skipping addresses it trusts, and stops at the first it doesn't. That's the real client.
- `req.hostname` reads `X-Forwarded-Host`.

**The security caveat — this is the whole reason the setting is not `true` by default:** `X-Forwarded-For` is just a request header. Any client can send it. If you set `trust proxy` to `true` (or to a hop count larger than the number of proxies you actually control), an attacker sends `X-Forwarded-For: <whatever they want>` and Express reports *that* as `req.ip`. Every downstream consumer is now poisoned:

- **Rate limiters** keyed on `req.ip` are trivially bypassed — send a new fake IP per request.
- **Audit logs** record attacker-chosen IPs.
- **IP allowlists / geo-blocks** are defeated.

The rule: **set `trust proxy` to exactly the number of proxy hops you operate, no more.** If exactly one load balancer sits in front of Node, `app.set('trust proxy', 1)` means "trust one hop from the socket." Express strips one entry off the right of `X-Forwarded-For` (the LB's own append) and takes the next as the client. If you have a CDN → LB → Node chain (two hops you control), use `2`. Getting this number wrong by one is a real, exploitable spoofing bug, not a cosmetic issue. See [X-Forwarded-For](../14-Proxies/X-Forwarded-For.md) and [Reverse Proxies](../16-Reverse-Proxies/) for the full threat model.

```js
// Behind exactly one trusted LB:
app.set('trust proxy', 1);

app.get('/whoami', (req, res) => {
  res.json({
    ip: req.ip,             // real client IP (LB hop stripped)
    proto: req.protocol,    // 'https' from X-Forwarded-Proto
    secure: req.secure,     // true
    host: req.hostname,     // from X-Forwarded-Host
  });
});
```

A common consequence you *want*: `res.cookie(..., { secure: true })` and any `secure`-conditional redirect logic depend on `req.secure` being accurate. If `trust proxy` is unset behind a TLS-terminating LB, `req.secure` is `false`, secure cookies are silently dropped, and you get an infinite redirect loop on "force HTTPS" middleware. See [Set-Cookie](../08-Cookies/Set-Cookie.md).

## Writing response headers

### `res.set()` / `res.header()` — the workhorse

`res.set()` sets a response header; `res.header()` is an identical alias. It accepts either a name/value pair or an object of many headers at once:

```js
res.set('Content-Type', 'application/json');

res.set({
  'Cache-Control': 'no-store',
  'X-Request-Id': req.get('X-Request-Id') ?? crypto.randomUUID(),
});
```

`res.set()` **overwrites** any existing value for that header. Two behaviors worth knowing:

- Setting `Content-Type` without a charset, for a value Express recognizes as text-like, causes Express to append `; charset=utf-8` automatically. Setting it explicitly with a charset disables that.
- Passing an array value (e.g. `res.set('Set-Cookie', ['a=1', 'b=2'])`) writes multiple header lines. This is the only clean way to emit repeated headers via `set`.

### `res.append()` — add without overwriting

`res.append()` adds a value to a header, keeping any existing value(s). This is the correct tool for headers that legitimately repeat or accumulate:

```js
res.append('Set-Cookie', 'session=abc');
res.append('Set-Cookie', 'theme=dark');   // now two Set-Cookie lines
res.append('Warning', '199 - "stale"');
```

If the header doesn't exist yet, `append` behaves like `set`. If it does, the value is coerced to an array and the new value pushed. Use `append` for `Set-Cookie`, `Vary`, `Link`, `Warning` — anything additive. Using `set` for these clobbers the earlier value, which is a classic "my second cookie disappeared" bug.

### `res.type()`, `res.location()`, `res.vary()` — targeted helpers

These are thin, purpose-built wrappers that encode a header's rules so you don't hand-roll them:

```js
res.type('json');          // Content-Type: application/json; charset=utf-8
res.type('.html');         // Content-Type: text/html; charset=utf-8
res.type('png');           // Content-Type: image/png
res.type('application/pdf');// passed through verbatim

res.location('/dashboard'); // Location: /dashboard  (does NOT set status)
res.location('back');       // Location: <Referer> or '/' — resolves the magic "back" value

res.vary('Accept-Encoding'); // adds to Vary WITHOUT duplicating if already present
res.vary('Origin');          // appends -> Vary: Accept-Encoding, Origin
```

Why `res.vary()` exists instead of `res.set('Vary', ...)`: `Vary` is a comma-separated *set*, and duplicating a field name in it (or clobbering existing entries) breaks caches. `res.vary()` reads the current value, dedupes, and appends. This is exactly the kind of header where the naive `set` is subtly wrong. See [Caching Headers](../06-Caching-Headers/) and [Content Negotiation](../11-Content-Negotiation/).

Note that `res.location()` sets *only* the `Location` header — it does **not** set a 3xx status. A redirect needs both; use `res.redirect()` (which calls `location()` + sets status + writes a body) or set the status yourself.

### `res.status()` and the status line

`res.status(code)` sets the numeric status. It's chainable and sets nothing else:

```js
res.status(201).json({ id: 42 });
res.status(304).end();              // conditional-request "not modified"
res.status(401).set('WWW-Authenticate', 'Bearer').end();
```

The status code lives on the **status line**, not in a header, but it's committed to the socket at the same moment the headers are — which is the crux of the next section.

### `res.cookie()` — Set-Cookie the ergonomic way

`res.cookie()` builds a `Set-Cookie` header from an options object, handling encoding and attribute serialization:

```js
res.cookie('session', token, {
  httpOnly: true,      // not readable by document.cookie — blocks XSS token theft
  secure: true,        // only sent over HTTPS (needs correct req.secure / trust proxy)
  sameSite: 'lax',     // CSRF mitigation; 'none' requires secure:true
  maxAge: 1000 * 60 * 60,  // ms in Express, serialized to Max-Age seconds
  path: '/',
  signed: false,       // true requires cookie-parser with a secret
});
```

Each `res.cookie()` call `append`s another `Set-Cookie` line, so multiple cookies coexist. `res.clearCookie(name, opts)` emits a `Set-Cookie` with an expired date — and the `path`/`domain` in the clear must match the original or the browser won't remove it (a frequent "logout doesn't clear the cookie" bug). Full semantics in [Set-Cookie](../08-Cookies/Set-Cookie.md).

## `res.writeHead` vs `setHeader`: timing and the headers-sent trap

This is the single most important mechanical concept in the chapter, because it explains a whole category of crashes.

An HTTP response is written to the socket in order: **status line, then headers, then a blank line, then body.** Once any byte of the body (or the head, via `writeHead`) has been flushed, the headers are **committed** — they're already on the wire and physically cannot be changed. Node tracks this with the boolean `res.headersSent`.

There are two ways headers get written:

- **`res.setHeader(name, value)`** (and Express's `res.set`, which wraps it) — *buffers* the header in memory. Nothing is sent yet. You can call it repeatedly, overwrite, remove.
- **`res.writeHead(status, headers)`** — writes the status line and all buffered-plus-passed headers to the socket **immediately**. After this, `res.headersSent` is `true`.

The body-writing calls (`res.write()`, `res.send()`, `res.json()`, `res.end()`) implicitly call `writeHead` if it hasn't been called yet. So the *first* thing that produces output commits the headers.

```js
app.get('/x', (req, res) => {
  res.setHeader('X-A', '1');   // buffered — fine
  res.send({ ok: true });      // commits headers (X-A included), writes body
  res.setHeader('X-B', '2');   // THROWS: ERR_HTTP_HEADERS_SENT
});
```

That last line throws:

```
Error [ERR_HTTP_HEADERS_SENT]: Cannot set headers after they are sent to the client
```

It's not a warning — it's a thrown error, and if unhandled it can crash the request (or the process, depending on your error handling). The mental model: **`res.send`/`res.json`/`res.end` are a point of no return for headers.** Everything header-related must happen *before* the first body write.

### Where this actually bites you

The bug is rarely as obvious as the example above. It shows up as:

1. **Two responses in one handler.** A guard clause sends an error, forgets to `return`, and execution falls through to a second `res.send`:

   ```js
   app.get('/user/:id', async (req, res) => {
     const user = await db.find(req.params.id);
     if (!user) {
       res.status(404).json({ error: 'not found' });
       // MISSING return — falls through
     }
     res.json(user); // second send: ERR_HTTP_HEADERS_SENT (and reads null.name)
   });
   ```
   Fix: `return res.status(404).json(...)`. Always `return` after you respond in a branch.

2. **Async work after `res.end()`.** You stream a response, call `res.end()`, then an awaited callback tries to set a trailing header.

3. **Middleware that sets a header late.** A logging or metrics middleware runs *after* the route in an unusual ordering and tries to `res.set(...)` on an already-committed response. See middleware ordering below.

Defensive pattern when you genuinely can't be sure (e.g. in error middleware that might run after a partial response):

```js
app.use((err, req, res, next) => {
  if (res.headersSent) {
    return next(err); // delegate to Node's default, which closes the connection
  }
  res.status(500).json({ error: 'internal' });
});
```

`res.headersSent` is the correct guard. Checking it is exactly what Express's own default error handler does, and it's why you should call `next(err)` (not try to respond) when it's `true`.

### `res.flushHeaders()` and streaming

For streaming / SSE / long-poll, you sometimes *want* to commit headers early — send the 200 + `Content-Type: text/event-stream` immediately so the client starts listening, then write body chunks over time. `res.flushHeaders()` forces the header commit without ending the response:

```js
res.status(200).set({
  'Content-Type': 'text/event-stream',
  'Cache-Control': 'no-cache',
  Connection: 'keep-alive',
});
res.flushHeaders();          // headers on the wire NOW; body stays open
const timer = setInterval(() => res.write(`data: ${Date.now()}\n\n`), 1000);
req.on('close', () => clearInterval(timer)); // client disconnected — stop writing
```

After `flushHeaders()`, `res.headersSent` is `true` and you're in body-only territory — same rules as after `send`.

## Removing headers and hiding the stack

### `res.removeHeader(name)`

Removes a *buffered* header before it's committed. Only works while `res.headersSent` is `false`:

```js
res.set('X-Debug', 'value');
res.removeHeader('X-Debug');  // gone, never sent
```

Express has no `res.unset()`; `removeHeader` is the raw Node method and it's the right one.

### Disabling `X-Powered-By`

Express, by default, advertises itself on every response:

```
X-Powered-By: Express
```

This is free reconnaissance for an attacker — it tells them your framework, which narrows the exploit search space. Turn it off globally:

```js
app.disable('x-powered-by');
// equivalent to: app.set('x-powered-by', false)
```

Do this on every production Express app. (Helmet also removes it — see below — so if you use Helmet you get this for free, but `app.disable('x-powered-by')` is the explicit one-liner.) It is not a strong defense on its own — security by obscurity — but there's zero cost and no reason to hand out the information.

## The header-family middleware

Express itself sets almost no headers beyond `Content-Type`, `Content-Length`, `ETag` (for some responses), and `X-Powered-By`. The heavy lifting is done by a small set of near-universal middleware, each owning a *family* of headers. Knowing which middleware owns which family — and in what order they must run — is most of what "configuring Express headers correctly" means.

### `helmet` — security headers

Helmet sets a bundle of security-related response headers with sensible defaults. What it does, header by header (defaults as of Helmet v7+):

```js
import helmet from 'helmet';
app.use(helmet());
```

| Header set | Purpose | Chapter |
|---|---|---|
| `Content-Security-Policy` | Restricts sources for scripts/styles/etc. — the big one | [Content-Security-Policy](../05-Security-Headers/Content-Security-Policy.md) |
| `Strict-Transport-Security` | Force HTTPS for future visits | [Strict-Transport-Security](../05-Security-Headers/Strict-Transport-Security.md) |
| `X-Content-Type-Options: nosniff` | Stop MIME sniffing | |
| `X-Frame-Options: SAMEORIGIN` | Clickjacking defense (legacy; CSP `frame-ancestors` supersedes) | |
| `Referrer-Policy` | Controls the `Referer` sent onward | [Referrer-Policy](../05-Security-Headers/Referrer-Policy.md) |
| `X-DNS-Prefetch-Control`, `X-Download-Options`, `Origin-Agent-Cluster`, `Cross-Origin-*` | Assorted hardening | |
| (removes) `X-Powered-By` | Reduce fingerprinting | |

Helmet's default CSP is deliberately strict and **will break** an app that uses inline scripts, inline styles, or CDN assets until you configure `directives`. Do not disable CSP to "make it work" — configure it:

```js
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", 'https://cdn.example.com'],
      styleSrc: ["'self'", "'unsafe-inline'"], // pragmatic if you can't nonce yet
      imgSrc: ["'self'", 'data:', 'https:'],
      connectSrc: ["'self'", 'https://api.example.com'],
    },
  },
  // HSTS: only enable once you're confident HTTPS is permanent
  strictTransportSecurity: { maxAge: 15552000, includeSubDomains: true },
}));
```

The nonce-based CSP pattern (for React/SPA inline bootstrap scripts) is covered in [Headers and React](../18-React/Headers-and-React.md) and [Content-Security-Policy](../05-Security-Headers/Content-Security-Policy.md).

### `cors` — cross-origin headers

The `cors` middleware sets the `Access-Control-*` response headers and, crucially, handles the **preflight `OPTIONS`** request. Its whole job is to translate a config object into the CORS negotiation described in [CORS Overview](../07-CORS/CORS-Overview.md).

```js
import cors from 'cors';

app.use(cors({
  origin: ['https://app.example.com'],   // reflect only these; NOT '*'
  credentials: true,                     // allow cookies/Authorization; forbids '*' origin
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  exposedHeaders: ['X-Total-Count'],     // what JS may READ off the response
  maxAge: 86400,                         // cache preflight 24h -> Access-Control-Max-Age
}));
```

Key facts that trip people up:

- **`credentials: true` is incompatible with `origin: '*'`.** The browser refuses a credentialed response whose `Access-Control-Allow-Origin` is the wildcard. The `cors` package handles this by *reflecting* the specific request origin when you pass an allowlist — but only if you gave it a concrete origin list or function, not `'*'`.
- **Exposed vs. allowed headers are different axes.** `allowedHeaders` = which *request* headers the client may send (answered in preflight). `exposedHeaders` = which *response* headers JS is permitted to read. By default JS can only read a handful of safelisted response headers; anything custom (`X-Total-Count`, `X-Request-Id`) is invisible to `fetch`/`axios` unless exposed. See [Access-Control-Expose-Headers](../07-CORS/Access-Control-Expose-Headers.md).
- `cors()` with no args is `origin: '*'`, no credentials — fine for a truly public read-only API, wrong for anything with auth.

### `compression` — Content-Encoding

`compression` gzip/brotli-compresses response bodies and sets `Content-Encoding` + `Vary: Accept-Encoding`:

```js
import compression from 'compression';
app.use(compression()); // reads client's Accept-Encoding, compresses if worthwhile
```

It negotiates via the request's `Accept-Encoding` and skips small bodies / already-compressed types. It **must run before** the handlers that generate the body (it wraps `res.write`/`res.end`), which is why it goes near the top. See [Compression](../10-Compression/).

Two production caveats: (1) If a CDN or reverse proxy already compresses, doing it again in Node is wasted CPU — pick one layer. (2) Compressing responses that reflect secret-length-correlated content over TLS is the BREACH attack vector; for sensitive endpoints (e.g. those echoing CSRF tokens) consider disabling compression.

### `cookie-parser` — reading cookies

`cookie-parser` parses the incoming `Cookie` header into `req.cookies` (and `req.signedCookies` when a secret is set). It reads a *request* header; it does not set response headers (that's `res.cookie`):

```js
import cookieParser from 'cookie-parser';
app.use(cookieParser(process.env.COOKIE_SECRET)); // secret enables signed cookies

app.get('/', (req, res) => {
  req.cookies.theme;          // 'dark'  (unsigned)
  req.signedCookies.session;  // verified value, or false if tampered
});
```

Signed cookies aren't encrypted — they're HMAC-signed so the server can detect tampering. The value is still readable by anyone. For secrets, encrypt or store server-side and reference by opaque ID. See [Set-Cookie](../08-Cookies/Set-Cookie.md).

### `morgan` — logging (and why it's last-ish)

`morgan` logs each request/response, typically reading `req.method`, `req.url`, `res.statusCode`, and response headers like `Content-Length`. It logs *after* the response finishes (it hooks `res.on('finish')`), so its position is mostly about *what state it observes*:

```js
import morgan from 'morgan';
app.use(morgan('combined')); // Apache combined log format
```

If you put `morgan` before `trust proxy` takes effect or before an ID-assigning middleware, it logs pre-transform values. Place it after `trust proxy` config and after any request-ID middleware so the logs record the real client IP (`req.ip`) and the correlation ID.

## Order of middleware: why it decides everything

Express middleware is a **linear pipeline**. `app.use(fn)` pushes `fn` onto an ordered stack; every request walks the stack top to bottom until something responds or `next()` runs out. Because response-header middleware works by *wrapping* `res` methods or *setting* headers before the body commits, order is not stylistic — it's semantic.

The rules that actually matter:

1. **Things that must see/modify the outgoing body wrap it, so they go first.** `compression` overrides `res.write`/`res.end`; it must be installed before any handler calls them. If it's registered after the route, it never sees the body.

2. **Security headers should apply to *every* response, including errors — so `helmet` goes early**, before routing. If Helmet is registered after your routes, a route that responds directly emits no CSP.

3. **CORS must run before anything that can respond, and before Helmet's cross-origin headers can conflict.** The preflight `OPTIONS` must be answered by `cors` before your auth middleware rejects it (an `OPTIONS` request has no credentials, so auth-first ordering makes the browser's preflight fail with 401 and the real request never fires). Put `cors` near the very top. The classic CORS+Helmet interaction: both touch `Cross-Origin-Resource-Policy`/`Cross-Origin-Opener-Policy`; if you serve cross-origin resources, Helmet's default `Cross-Origin-Resource-Policy: same-origin` will block them even though `cors` allowed the fetch — you must relax that Helmet directive, not just configure `cors`.

4. **Body-independent request enrichment (cookie-parser, body parsers, request-ID) goes before routes** so handlers see populated `req.cookies`, `req.body`, etc.

5. **Error-handling middleware (4-arg signature) goes last**, after routes, because Express only routes to it via `next(err)`.

6. **`trust proxy` is an app setting, not middleware** — set it before the server starts listening; it's not order-sensitive relative to `use()` calls, but every IP/proto-dependent middleware reads it, so establish it up front.

A wrong order that "works" in dev and breaks in prod: registering `helmet` *after* the static file middleware. In dev you hit routes that run through Helmet; the static assets (served by `express.static` which responds immediately) never get CSP — and you don't notice until a pentest flags missing headers on your JS bundles.

## A complete, annotated production `app.js`

This wires every family together in the order that satisfies the rules above. Every non-obvious line is annotated with *why* and *what breaks without it*.

```js
import express from 'express';
import helmet from 'helmet';
import cors from 'cors';
import compression from 'compression';
import cookieParser from 'cookie-parser';
import morgan from 'morgan';
import crypto from 'node:crypto';

const app = express();

// --- App-level settings (not middleware; establish before listening) ---

// We run behind exactly ONE reverse proxy / load balancer that terminates TLS
// and appends to X-Forwarded-For. Trusting exactly 1 hop makes req.ip the real
// client and req.secure accurate. Setting this to `true` would let any client
// spoof X-Forwarded-For and defeat rate limiting / IP logging. See ../14-Proxies.
app.set('trust proxy', 1);

// Do not advertise "X-Powered-By: Express" — free recon for attackers.
// (Helmet also removes it; this is the explicit belt-and-suspenders.)
app.disable('x-powered-by');

// --- 1. Request correlation ID (must be first so EVERYTHING downstream, ---
// ---    including logs and error responses, shares the id) ---
app.use((req, res, next) => {
  // Reuse an upstream id if the proxy/CDN set one, else mint a new one.
  const id = req.get('X-Request-Id') ?? crypto.randomUUID();
  req.id = id;
  res.set('X-Request-Id', id); // echo it so clients can correlate. Buffered, not sent yet.
  next();
});

// --- 2. Logging (after id + trust proxy so it records real client ip and id) ---
// morgan logs on res 'finish', so it observes the FINAL status/length.
morgan.token('id', (req) => req.id);
app.use(morgan(':id :remote-addr :method :url :status :res[content-length] - :response-time ms'));

// --- 3. CORS — BEFORE auth and before Helmet's cross-origin blocks, and it ---
// ---    must answer the preflight OPTIONS itself. ---
app.use(cors({
  origin: (origin, cb) => {
    const allow = ['https://app.example.com', 'https://admin.example.com'];
    // Allow same-origin / server-to-server (no Origin header) and allowlisted origins.
    if (!origin || allow.includes(origin)) return cb(null, true);
    return cb(new Error('Not allowed by CORS'));
  },
  credentials: true,            // needed so the browser sends cookies / Authorization.
  // credentials:true means A-C-A-Origin is REFLECTED, never '*'. Do not set '*' here.
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Request-Id'],
  exposedHeaders: ['X-Request-Id', 'X-Total-Count'], // else JS can't read these
  maxAge: 86400,                // cache preflight 24h; fewer OPTIONS round-trips.
}));

// --- 4. Security headers for EVERY response (including static + errors) ---
app.use(helmet({
  contentSecurityPolicy: {
    // Generate a per-request nonce so inline bootstrap <script nonce> is allowed
    // WITHOUT 'unsafe-inline'. Expose it to templates via res.locals.
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", (req, res) => `'nonce-${res.locals.cspNonce}'`],
      styleSrc: ["'self'"],
      connectSrc: ["'self'", 'https://api.example.com'],
      imgSrc: ["'self'", 'data:'],
      objectSrc: ["'none'"],
      baseUri: ["'self'"],
      frameAncestors: ["'none'"],   // clickjacking defense; supersedes X-Frame-Options
    },
  },
  strictTransportSecurity: {
    maxAge: 15552000,               // 180 days. Only ship once HTTPS is permanent —
    includeSubDomains: true,        // this is hard to undo; browsers pin it.
  },
  // If you serve assets consumed cross-origin, you'd relax this; default blocks it:
  crossOriginResourcePolicy: { policy: 'same-origin' },
}));

// Mint the nonce BEFORE Helmet reads res.locals.cspNonce above — so this middleware
// must run before helmet(). (Shown after for readability; in real code order it first.)
app.use((req, res, next) => {
  res.locals.cspNonce = crypto.randomBytes(16).toString('base64');
  next();
});

// --- 5. Compression — must wrap res.write/res.end, so BEFORE routes ---
app.use(compression({
  // Let clients opt out with a header (handy for debugging / SSE):
  filter: (req, res) => req.get('x-no-compression') ? false : compression.filter(req, res),
}));

// --- 6. Body + cookie parsing — populate req.body / req.cookies before routes ---
app.use(express.json({ limit: '100kb' }));           // sets nothing on res; parses Content-Type: application/json
app.use(express.urlencoded({ extended: true }));
app.use(cookieParser(process.env.COOKIE_SECRET));    // req.cookies / req.signedCookies

// --- 7. Static assets: hashed bundles cached hard, index.html never cached ---
app.use('/assets', express.static('public/assets', {
  immutable: true,
  maxAge: '1y',        // hashed filenames -> safe to cache forever. See ../06-Caching-Headers.
}));

// --- 8. Application routes ---
app.get('/api/health', (req, res) => {
  res.set('Cache-Control', 'no-store'); // health checks must never be cached
  res.json({ ok: true, id: req.id });
});

app.post('/api/login', (req, res) => {
  const token = signSession(req.body); // your auth
  res.cookie('session', token, {
    httpOnly: true,                     // JS can't read it -> XSS can't steal it
    secure: req.secure,                 // true because trust proxy makes req.secure accurate
    sameSite: 'lax',                    // CSRF mitigation
    maxAge: 1000 * 60 * 60 * 8,
    path: '/',
  });
  res.status(204).end();
});

app.get('/api/items', async (req, res) => {
  const { rows, total } = await queryItems(req.query);
  res.set('X-Total-Count', String(total)); // exposed via CORS above, so JS can read it
  res.vary('Accept-Encoding');             // dedup-safe; compression also relies on Vary
  res.json(rows);                          // <-- FIRST body write; headers commit here.
  // Anything header-related after this line throws ERR_HTTP_HEADERS_SENT.
});

// --- 9. 404 (after all routes, before error handler) ---
app.use((req, res) => {
  res.status(404).json({ error: 'not_found', id: req.id });
});

// --- 10. Error handler — 4 args; MUST be last. Guards headersSent. ---
app.use((err, req, res, next) => {
  if (res.headersSent) {
    // Response already committed (e.g. error mid-stream). Can't set status/headers now;
    // hand off to Node's default handler, which destroys the socket.
    return next(err);
  }
  const status = err.status ?? 500;
  res.status(status).json({ error: err.message, id: req.id });
});

app.listen(3000, () => console.log('listening on :3000'));
```

The ordering caveat inside the example (nonce middleware shown after `helmet` for readability, but must run before it) is itself the lesson: Helmet's CSP function reads `res.locals.cspNonce` at response time, so the middleware that *sets* the nonce must appear earlier in the stack. In real code, register the nonce middleware immediately before `helmet()`.

## Common mistakes

- **Setting a header after `res.end()` / `res.send()`.** `ERR_HTTP_HEADERS_SENT`. Everything header-related must precede the first body write. Guard with `res.headersSent` where the flow is uncertain, and always `return` after responding in a branch.

- **CORS + Helmet ordering / cross-origin resource policy.** Registering `helmet` before `cors`, or leaving Helmet's default `Cross-Origin-Resource-Policy: same-origin` while trying to serve assets cross-origin, blocks resources the CORS config appeared to allow. Put `cors` first; relax the specific Helmet directive if you serve cross-origin. Also: auth middleware before `cors` kills the credential-less preflight `OPTIONS` with a 401.

- **`trust proxy` misconfiguration.** `true` (or too many hops) behind a proxy lets clients spoof `X-Forwarded-For`, defeating rate limits and poisoning IP logs. Unset behind a TLS-terminating proxy makes `req.secure` false, dropping `secure` cookies and causing HTTPS-redirect loops. Set it to *exactly* your controlled hop count.

- **`res.set('Set-Cookie', ...)` clobbering an earlier cookie.** Use `res.cookie()` / `res.append()` for repeated headers; `set` overwrites.

- **Compression after the route** (never sees the body), or double compression when a CDN already gzips (wasted CPU, and a BREACH vector for secret-echoing endpoints).

- **Missing `res.vary('Accept-Encoding')`** on content-negotiated responses, letting a shared cache serve a gzipped body to a client that can't decode it. `compression` sets `Vary` for you, but hand-negotiated responses need it explicitly. See [Content Negotiation](../11-Content-Negotiation/).

- **Reading `req.headers['Authorization']`** (capital A) and getting `undefined` because Node lowercases keys. Use `req.get('Authorization')`.

- **Leaving `X-Powered-By: Express`** on. `app.disable('x-powered-by')` — free, do it.

## Mental Model

Think of an Express response as a train leaving a station. **Everything about the train — the destination sign (status), the labels on the cars (headers) — must be finalized before it departs.** Departure is the *first byte of body*: `res.send`, `res.json`, `res.write`, `res.end`, or an explicit `res.writeHead`/`res.flushHeaders`. `setHeader`/`res.set` are you writing on the cars while the train sits at the platform (buffered, changeable). Once it rolls (`headersSent === true`), the cars are gone — any attempt to relabel throws `ERR_HTTP_HEADERS_SENT`.

The `req` side is the mirror image: Express hands you *interpreted* views (`req.ip`, `req.secure`, `req.hostname`, `req.protocol`) that are only as trustworthy as your `trust proxy` setting, because they're derived from headers a client could forge. Treat them as trustworthy exactly to the extent you've told Express how many proxies you actually control — no more.

And the middleware stack is a **pipeline of header-family owners** running in a fixed order: `cors` opens the cross-origin door (and answers preflights) first, `helmet` stamps security headers on everything, `compression` wraps the body writer, the parsers enrich `req`, your routes set the response-specific headers and commit the train, and the error handler cleans up only if the train hasn't left. Get the order wrong and a later car overwrites an earlier one, or the security stamp never gets applied — not because any single call was wrong, but because the pipeline ran them in the wrong sequence.
