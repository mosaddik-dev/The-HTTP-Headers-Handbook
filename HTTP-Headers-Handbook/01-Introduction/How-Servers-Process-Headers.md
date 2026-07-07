# How Servers Process Headers

On the browser side, a security-conscious user agent owns the header namespace (see [How Browsers Process Headers](./How-Browsers-Process-Headers.md)). On the server side the situation inverts: *you* own the response header namespace almost completely, and the raw request headers arrive largely unfiltered — which means normalization, validation, and framing decisions become *your* responsibility. This chapter follows a request from raw bytes through Node's HTTP parser into `req.headers`, then up through Express's convenience layer, and finally back out as a response with correctly framed bytes.

The audience assumption throughout: you know Node and Express. What you may not know is exactly *where* the lowercasing happens, *why* `req.rawHeaders` exists, what `--max-http-header-size` protects against, and how middleware ordering can silently strip a header you thought you set.

## From bytes to `req.headers`: Node's HTTP parser

Node's `http` and `https` servers do not parse HTTP in JavaScript. They delegate to **llhttp**, a C parser compiled into the binary, which reads the incoming byte stream, identifies the request line and header block, and emits parsed fields up to the JS layer. By the time your request handler runs, llhttp has produced two representations on the `IncomingMessage` (`req`) object.

### `req.headers` — lowercased, merged, ready to use

`req.headers` is a plain object where **every header name is lowercased** and, for most headers, duplicate occurrences are merged. This is the representation you use 99% of the time.

```js
import http from "node:http";

const server = http.createServer((req, res) => {
  // Case-insensitive by construction — the key is always lowercase.
  console.log(req.headers["content-type"]); // "application/json"
  console.log(req.headers["x-custom"]);      // whatever the client sent

  res.writeHead(200, { "Content-Type": "application/json" });
  res.end(JSON.stringify({ ok: true }));
});

server.listen(3000);
```

Why lowercase? HTTP/1.1 header names are case-insensitive, and HTTP/2 mandates lowercase on the wire (see [HTTP Versions and Headers](./HTTP-Versions-and-Headers.md)). Node normalizes to lowercase so your code never has to guess whether the client sent `Content-Type`, `content-type`, or `CONTENT-TYPE`. **Always index `req.headers` with lowercase keys** — `req.headers["Content-Type"]` is `undefined`.

### The merge rules are not uniform

How Node combines duplicate headers depends on the header, and getting this wrong causes real bugs:

- **Most headers**: a duplicate *discards* all but the first occurrence. If a client sends `Content-Type` twice, you get the first only.
- **These specific headers are comma-joined** when repeated: `age`, `authorization`, `content-length`, `content-type`, `etag`, `expires`, `from`, `host`, `if-modified-since`, `if-unmodified-since`, `last-modified`, `location`, `max-forwards`, `proxy-authorization`, `referer`, `retry-after`, `server`, `user-agent` — actually these are the *discard-duplicates* set. The precise rule: a small allowlist is comma-joined, `set-cookie` is *always an array*, and the rest discard duplicates.
- **`set-cookie`**: always exposed as an **array** (`req.headers["set-cookie"]` → `string[]`), because folding multiple cookies into one comma-separated value would corrupt them (cookie values legally contain commas).
- **Cookie (request)**: multiple `Cookie` lines are joined with `; `.

The upshot: if you need byte-exact fidelity — proxies, signature verification, debugging a client that sends odd duplicates — `req.headers` is *lossy*. That is what `req.rawHeaders` is for.

### `req.rawHeaders` — the unmodified, ordered pairs

`req.rawHeaders` is a flat array of `[name, value, name, value, ...]` preserving **original case, original order, and every duplicate**. Even-indexed entries are names; odd-indexed are values.

```js
import http from "node:http";

const server = http.createServer((req, res) => {
  const raw = req.rawHeaders; // e.g. ["Host","x.com","Accept","*/*","X-Dup","a","X-Dup","b"]

  // Reconstruct all values for a header, case-insensitively, preserving duplicates.
  function getAll(name) {
    const target = name.toLowerCase();
    const out = [];
    for (let i = 0; i < raw.length; i += 2) {
      if (raw[i].toLowerCase() === target) out.push(raw[i + 1]);
    }
    return out;
  }

  res.writeHead(200, { "content-type": "application/json" });
  res.end(JSON.stringify({
    merged: req.headers["x-dup"],   // "a" (duplicate discarded, or comma-joined per rules)
    raw: getAll("x-dup"),           // ["a", "b"] — both preserved
    order: raw.filter((_, i) => i % 2 === 0), // exact on-wire order & casing
  }));
});

server.listen(3000);
```

Use `rawHeaders` when order or exact casing is semantically meaningful: reverse proxies that must forward headers verbatim, HTTP signature schemes that sign a canonicalized-but-ordered header list, or forensic logging. For everything else, `req.headers` is correct and simpler.

## Header size limits, `--max-http-header-size`, and 431

llhttp enforces a hard cap on the total size of the header block. The default is **16 KB** (16384 bytes) across all headers combined. This is a denial-of-service guard: without it, a client could stream an unbounded header block and exhaust server memory before any request logic runs.

When a client exceeds the limit, Node does not hand you a request — it responds (or the parser errors) with **`431 Request Header Fields Too Large`**, or drops the connection. You cannot catch this in a normal route handler because your handler never runs; the parser rejects the request first.

You adjust the ceiling at process start:

```bash
# Raise the header limit to 32 KB (e.g. behind an SSO proxy that injects large JWTs)
node --max-http-header-size=32768 server.js
```

```js
// Or per-server, programmatically:
import http from "node:http";
const server = http.createServer({ maxHeaderSize: 32768 }, handler);
```

When you hit a real-world `431`, it is almost always **oversized cookies** (accumulated tracking/session cookies) or a **large bearer token / SSO header** pushed by an upstream proxy. Raising the limit is a band-aid; the fix is usually trimming cookies or moving state out of headers. Note the security tradeoff: a higher limit is a larger memory-amplification surface, so raise it deliberately, not reflexively.

## The Express layer: `req.get()` and `res.set()`

Express does not re-parse anything. It wraps Node's `req`/`res` and adds convenience methods that read from and write to the same underlying header structures.

### Reading: `req.get()` / `req.header()`

`req.get(name)` (alias `req.header(name)`) is a case-insensitive lookup into `req.headers`, with two special cases baked in for the two most-referenced headers:

```js
app.use((req, res, next) => {
  req.get("content-type");  // same as req.headers["content-type"]
  req.get("Content-Type");  // case-insensitive — also works
  req.get("referer");       // "referer" and "referrer" are aliased here
  next();
});
```

It is thin sugar. `req.headers` still exists underneath and is what everything else in the ecosystem reads.

### Writing: `res.set()` / `res.append()` / `res.header()`

`res.set(field, value)` sets a response header on the underlying `res`. `res.append()` adds a value without clobbering an existing one (important for `Set-Cookie` and `Vary`). Express also provides higher-level setters — `res.type()` sets `Content-Type`, `res.cookie()` composes `Set-Cookie` with its attributes, `res.vary()` appends to `Vary` idempotently.

```js
app.get("/report.csv", (req, res) => {
  res.set("Content-Type", "text/csv; charset=utf-8"); // controls how the browser interprets the body
  res.set("Content-Disposition", 'attachment; filename="report.csv"'); // forces download vs inline render
  res.set("Cache-Control", "no-store"); // keep sensitive report out of shared/browser caches
  res.append("Vary", "Accept-Encoding"); // don't clobber a Vary another middleware may have set
  res.send("id,total\n1,42\n");
});
```

If you remove the `Content-Type` line, Express infers `text/html` from `res.send` with a string, and the browser tries to render CSV as a web page. If you use `res.set("Vary", ...)` instead of `res.append`, you overwrite a `Vary: Accept-Encoding` that compression middleware set, and shared caches may serve a gzip'd body to a client that cannot decode it. Each line has a failure mode.

### The one hard rule: headers must be set before the body flushes

HTTP sends the entire header block *before* the body. Node buffers response headers until the first body write (or `res.end`), at which point they are flushed and **locked**. Any `res.set()` after that throws `ERR_HTTP_HEADERS_SENT`.

```js
app.get("/x", (req, res) => {
  res.send("hi");             // flushes headers + body
  res.set("X-Late", "nope");  // throws: Cannot set headers after they are sent to the client
});
```

This is *the* reason middleware ordering matters for response headers — a header-setting middleware that runs after the handler already responded is a no-op at best and a crash at worst.

## Middleware ordering and its effect on headers

Middleware is a linear pipeline. Ordering determines both which request headers are *available* to later middleware and whether response headers land *before the body flushes*.

```mermaid
flowchart LR
    A[Incoming request] --> B[helmet - sets security response headers]
    B --> C[cors - reads Origin, sets ACAO/ACAC]
    C --> D[compression - sets Content-Encoding, Vary]
    D --> E[body parser - reads Content-Type]
    E --> F[route handler - res.send flushes]
```

Concrete ordering pitfalls:

- **`helmet` / security headers must come before the handler.** They set response headers (`Content-Security-Policy`, `X-Content-Type-Options`, `Strict-Transport-Security`). If registered *after* the route that sends the body, they run too late — headers are already flushed — and silently do nothing (or throw).
- **`cors` must run before the route** and, for preflight, must be able to short-circuit the `OPTIONS` request with the right `Access-Control-*` headers before any handler responds.
- **`compression` sets `Content-Encoding` and must append `Vary: Accept-Encoding`.** It has to wrap the response stream before the handler writes the body. Placed after, it cannot compress anything.
- **Body parsers read request headers** (`express.json()` checks `Content-Type: application/json`). They do not set response headers, so their ordering is about *reading* the request, but they must run before any handler that reads `req.body`.

The mental split: **request-reading middleware** must run before whoever consumes the parsed request; **response-header-writing middleware** must run before whoever flushes the body. Violate the first and `req.body`/`req.get()` is empty; violate the second and your header is dropped or throws.

## How the server frames the body: Content-Length vs. chunked

A response's *headers* tell the client where the body ends. There are two framing mechanisms, and the server chooses one automatically based on how you write the response.

- **`Content-Length`**: the server knows the exact byte count up front. When you call `res.send(buffer)` or `res.send(string)`, Node computes the byte length and sets `Content-Length` for you. The client reads exactly that many bytes and then knows the message is complete.
- **`Transfer-Encoding: chunked`**: the server does *not* know the total size in advance (streaming, generated-on-the-fly output). When you `res.write()` incrementally without a `Content-Length`, Node switches to chunked framing — each chunk is prefixed with its size in hex, and a zero-length chunk terminates the body.

```js
import http from "node:http";

http.createServer((req, res) => {
  if (req.url === "/known") {
    const body = JSON.stringify({ ok: true });
    // Node sets Content-Length automatically here because the full body is known.
    res.writeHead(200, { "Content-Type": "application/json" });
    res.end(body);
  } else {
    // No Content-Length set → Node uses Transfer-Encoding: chunked.
    res.writeHead(200, { "Content-Type": "text/plain" });
    res.write("first chunk\n");
    setTimeout(() => { res.write("second chunk\n"); res.end(); }, 50);
  }
}).listen(3000);
```

The corresponding wire output for the streaming case:

```http
HTTP/1.1 200 OK
Content-Type: text/plain
Transfer-Encoding: chunked

c
first chunk

d
second chunk

0

```

Two rules the server enforces:

1. **You must not set both `Content-Length` and `Transfer-Encoding: chunked`.** They are mutually exclusive framing schemes, and sending both is the classic **request smuggling** vector — a downstream proxy and an origin server disagreeing on which one wins lets an attacker inject a second request into the stream. Node will strip/error on the conflict; never set both by hand.
2. On **HTTP/2 and HTTP/3, `Transfer-Encoding: chunked` is banned entirely** — framing is handled by the protocol's own binary framing layer, and body length is conveyed differently. See [HTTP Versions and Headers](./HTTP-Versions-and-Headers.md). Node manages this for you when serving over h2, but hand-rolled framing headers will be rejected.

This framing concern is exactly the hop-by-hop vs. end-to-end distinction from [End-to-End vs Hop-by-Hop Headers](./End-to-End-vs-Hop-by-Hop-Headers.md): `Transfer-Encoding` and `Connection` are hop-by-hop and must be re-decided at every connection, which is why they cannot survive a protocol change.

## Validation and normalization on the server

Because raw request headers arrive from an untrusted client, the server (and you) must treat them as input, not truth.

- **llhttp rejects malformed headers** at parse time: header names with illegal characters, missing colons, or bare CR/LF are rejected before your code runs. This is a security boundary — it is what prevents header injection via a smuggled newline. Node's `lenient` mode can relax some of this, but do not enable it without reason.
- **Response header values are validated too.** If you try `res.set("X-Foo", "bad\r\nInjected: header")`, Node throws `ERR_INVALID_CHAR`, because a raw CRLF in a header value would let an attacker inject additional headers or split the response (CRLF/response-splitting). Always validate/encode any user-derived value before putting it in a response header.
- **Trust, but verify, proxy headers.** `X-Forwarded-For`, `X-Forwarded-Proto`, and `Host` are frequently spoofable unless you sit behind a trusted proxy. Express's `app.set("trust proxy", ...)` tells Express which upstreams to trust when interpreting these — misconfiguring it means `req.ip` and `req.protocol` can be forged by a client. Never make a security decision on a forwarded header without a configured trust boundary.

## Mental Model

Think of the server as the **inverse of the browser gatekeeper**, but with the trust boundary reversed.

- **Inbound**, Node's C parser (llhttp) does the heavy lifting *before your JS runs*: it enforces the header-size cap (16 KB → `431`), rejects malformed/injected headers, and produces two views — `req.headers` (lowercased, merged, convenient, lossy) and `req.rawHeaders` (verbatim, ordered, complete). Reach for `rawHeaders` only when order or duplicates carry meaning.
- **In the middle**, Express is thin sugar over those same structures. `req.get()` reads `req.headers`; `res.set()` writes the underlying response headers. Middleware ordering is not stylistic — request-reading middleware must precede its consumer, and response-header-writing middleware must precede the body flush, after which headers lock and any write throws `ERR_HTTP_HEADERS_SENT`.
- **Outbound**, you own the response headers, and the server chooses body framing from *how you write*: full body → `Content-Length`; incremental stream → `Transfer-Encoding: chunked` (never both — that is smuggling; and never chunked on h2/h3).

The recurring discipline: request headers are **untrusted input** you normalize and validate; response headers are **your output** where a stray `\r\n` becomes a vulnerability. The parser guards the former; you guard the latter.

Next: [HTTP Versions and Headers](./HTTP-Versions-and-Headers.md) — how the same header model is carried very differently across HTTP/1.1, /2, and /3.
