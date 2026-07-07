# Anatomy of an HTTP Message

## Quick Summary

Every HTTP message — request or response — has the same four-part skeleton: a **start line**, a block of **headers**, an empty line (`CRLF`), and an optional **body**. Headers live between the start line and the body, one per line, in `Name: value` form. Understanding exactly where headers sit on the wire, how they're delimited, and how the parser finds the boundary between headers and body is the foundation for everything else in this handbook — including why request smuggling, response splitting, and header-injection attacks are possible.

## The four parts

```
┌─────────────────────────────────────────────┐
│ start line     GET /index.html HTTP/1.1      │  ← request line OR status line
├─────────────────────────────────────────────┤
│ header field   Host: example.com             │
│ header field   Accept: text/html             │  ← zero or more headers
│ header field   Accept-Encoding: gzip         │
├─────────────────────────────────────────────┤
│ empty line     (CRLF)                         │  ← the separator that ends headers
├─────────────────────────────────────────────┤
│ body           <html>...</html>              │  ← optional payload
└─────────────────────────────────────────────┘
```

This structure is defined by RFC 9112 (HTTP/1.1 message syntax). It is deliberately line-oriented and human-readable in HTTP/1.x, which is why you can literally type an HTTP request into a raw TCP socket.

## Part 1 — The start line

The start line differs between requests and responses.

**Request line** = method + request-target + HTTP-version:

```http
GET /api/users/42?expand=profile HTTP/1.1
```

- **Method** (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD`, `OPTIONS`, …) — the action.
- **Request-target** — usually the path + query. (For proxies it can be an absolute URL; for `CONNECT` it's `host:port`; for `OPTIONS *` it's `*`.)
- **HTTP-version** — `HTTP/1.1`, etc.

**Status line** = version + status code + reason phrase:

```http
HTTP/1.1 404 Not Found
```

The reason phrase (`Not Found`) is purely informational and is *dropped entirely* in HTTP/2 and HTTP/3 — never parse it programmatically.

## Part 2 — The header section

After the start line come the header fields, each terminated by `CRLF` (`\r\n`):

```http
Host: example.com\r\n
Content-Type: application/json\r\n
Content-Length: 27\r\n
```

Grammar essentials (from RFC 9110 §5):

- A header field is `field-name ":" OWS field-value OWS` where `OWS` is optional whitespace.
- **Field names are case-insensitive** tokens: `Content-Type` ≡ `content-type`.
- The **colon must immediately follow the name** — `Host : x` (space before colon) is illegal and a conforming server must reject or ignore it. This strictness is a security feature (see request smuggling below).
- Whitespace *around* the value is trimmed; whitespace *inside* the value is preserved.

### Multiple values and repeated fields

Two legal ways to express multiple values:

```http
Accept-Encoding: gzip, br, deflate
```
or
```http
Accept-Encoding: gzip
Accept-Encoding: br
```

For most headers these are equivalent — a recipient may combine repeated fields into one comma-separated list. The **critical exception is `Set-Cookie`**, which must never be folded onto one line (its value legitimately contains commas in `Expires` dates). This is why Node exposes `res.getHeaders()` with `set-cookie` as an *array* while other headers are strings, and why HTTP/2 keeps each `set-cookie` as a separate field.

### Obsolete line folding

Historically a value could wrap across lines by starting the continuation with whitespace. This "obs-fold" is now **deprecated and dangerous** — modern parsers reject it because it enabled smuggling. Never rely on it.

## Part 3 — The empty line (the most important line)

A single empty `CRLF` marks the end of the header section:

```
...last header\r\n
\r\n            ← this blank line ends the headers; body (if any) follows
```

This is the boundary the parser uses to know "headers are done, everything after is body." Get this delimiter wrong — or let user input inject a `\r\n\r\n` into a header value — and you have **HTTP response splitting**: an attacker terminates your headers early and injects their own headers/body. This is exactly why frameworks forbid `\r`/`\n` in header values (Node throws `ERR_INVALID_CHAR` if you try `res.setHeader('X-Foo', 'a\r\nSet-Cookie: evil')`).

## Part 4 — The body

The body is optional and its presence/length is governed by headers, not by the body itself. There are three ways the recipient knows where the body ends:

1. **`Content-Length: N`** — exactly N bytes follow. Deterministic and required when you know the size.
2. **`Transfer-Encoding: chunked`** — the body arrives in length-prefixed chunks terminated by a zero-length chunk. Used for streaming when the total size isn't known up front (SSE, dynamically generated responses).
3. **Connection close** (HTTP/1.0 style) — body runs until the socket closes. Fragile; avoid.

**You must never send both a `Content-Length` and `Transfer-Encoding: chunked`.** When a message has both — or when a front-end proxy and back-end server disagree on which to honor — you get **HTTP request smuggling**, one of the most serious classes of web vulnerabilities. RFC 9112 says `Transfer-Encoding` wins and `Content-Length` must be treated as an error, but not all implementations agree, which is the whole attack. See [Transfer-Encoding](../10-Compression/Transfer-Encoding.md).

Some responses have no body by rule regardless of headers: `204 No Content`, `304 Not Modified`, and any response to a `HEAD` request. A `HEAD` response *carries* the `Content-Length` that the equivalent `GET` would have, but sends zero body bytes.

## Putting it together: a raw exchange

You can watch the exact bytes with `curl`. Note the blank line separating headers from body in both directions:

**Request (what leaves the client)**

```http
POST /api/login HTTP/1.1
Host: app.example.com
Content-Type: application/json
Content-Length: 44
Accept: application/json

{"email":"a@example.com","password":"hunter2"}
```

**Response (what comes back)**

```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 39
Set-Cookie: sid=abc123; HttpOnly; Secure; SameSite=Lax
Cache-Control: no-store

{"token":"eyJ...","user":{"id":42}}
```

The `Content-Length: 44` tells the server to read exactly 44 bytes of JSON. The response's blank line ends the headers; the 39-byte JSON body follows.

## Inspecting the raw bytes yourself

```bash
# Show the full request AND response headers, plus the boundary, verbosely:
curl -v https://app.example.com/api/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"a@example.com","password":"hunter2"}'

#   > lines  = bytes curl SENT (request line + request headers)
#   < lines  = bytes curl RECEIVED (status line + response headers)
#   a blank  < line marks the end of response headers

# Response headers only (sends a HEAD request):
curl -I https://example.com

# Force raw HTTP/1.1 and dump everything including timing:
curl -sv --http1.1 https://example.com -o /dev/null
```

In Node you can see the parser's view directly:

```js
const http = require('node:http');

http.createServer((req, res) => {
  // req.method / req.url  -> parsed from the request line
  console.log(req.method, req.url, `HTTP/${req.httpVersion}`);

  // req.rawHeaders -> flat array [name, value, name, value, ...] AS RECEIVED
  //   (preserves original casing and duplicate fields; nothing merged)
  console.log(req.rawHeaders);

  // req.headers -> lowercased object with duplicates merged (except set-cookie)
  console.log(req.headers);

  res.writeHead(200, { 'Content-Type': 'text/plain' }); // writes status line + headers + blank line
  res.end('ok'); // writes the body; end() finalizes Content-Length or chunked framing
}).listen(3000);
```

`req.rawHeaders` vs `req.headers` is the clearest illustration of the difference between *the bytes on the wire* and *the framework's normalized view*. When debugging weird header behavior, reach for `rawHeaders`.

## Why this anatomy matters in production

- **Framing bugs = security bugs.** Smuggling and splitting both come from ambiguity about where headers end and bodies begin. Knowing the anatomy tells you why frameworks are strict.
- **Header size limits exist here.** Servers cap the total header section size (Node default ~16KB via `--max-http-header-size`; Nginx `large_client_header_buffers`). Overflowing it yields `431 Request Header Fields Too Large`. Fat cookies and long `Authorization` tokens are the usual culprits.
- **The blank line is load-bearing.** Any code that constructs HTTP by hand (rare, but happens in proxies and tests) must emit exactly one `CRLF` between headers and body.

## Related Reading

- [Request vs Response Headers](./Request-vs-Response-Headers.md)
- [HTTP Versions and Headers](./HTTP-Versions-and-Headers.md) — how this text framing becomes binary in HTTP/2/3.
- [Transfer-Encoding](../10-Compression/Transfer-Encoding.md) and [Content-Length](../03-Request-Headers/Content-Length.md) — body framing in depth.
- [Header Size Limits](../02-Core-Concepts/Header-Size-Limits.md)

## Mental Model

**An HTTP message is a letter.** The start line is the "To/subject." The headers are the envelope annotations. The blank line is the fold that separates envelope from letter. The body is the letter itself. Machines read the envelope to route and handle the mail; the blank line is how they know the envelope has ended and the letter has begun. Smear ink across that fold and the mail room can't tell envelope from contents — which is precisely the attack.
