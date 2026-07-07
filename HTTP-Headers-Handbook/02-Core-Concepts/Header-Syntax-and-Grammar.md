# Header Syntax and Grammar

Most of the header bugs that reach production are not conceptual — they are *parsing* bugs. Someone splits `Accept` on the wrong comma, treats a quoted-string as opaque bytes, joins `Set-Cookie` into a single line, lowercases a value that was case-sensitive, or trusts that "the framework handles it." The HTTP specs pin all of this down precisely with ABNF grammar, and knowing that grammar is the difference between writing a header parser that works and one that fails on the first weird-but-legal input.

This chapter covers the field grammar from RFC 9110 §5 (Fields) and the message syntax from RFC 9112 (HTTP/1.1): the shape of a field line, tokens vs quoted-strings, optional whitespace (OWS), the ABNF list operator (`#rule`), parameters and q-values, how repeated fields combine, why line folding is dead, why `Set-Cookie` is the exception to nearly every rule, and the one date format you should ever emit. It closes with a correct, production-grade q-value list parser in Node.

## The shape of a field line

In HTTP/1.1 (RFC 9112 §5), a header field on the wire is:

```abnf
field-line   = field-name ":" OWS field-value OWS
field-name   = token
field-value  = *field-content
field-content = field-vchar [ 1*( SP / HTAB / field-vchar ) field-vchar ]
field-vchar  = VCHAR / obs-text
OWS          = *( SP / HTAB )   ; "optional whitespace"
```

Read that carefully, because several production rules fall straight out of it:

- **The colon has no space before it.** `Content-Type :` is malformed. RFC 9112 explicitly says a server MUST reject a request where whitespace appears between the field name and the colon (this exact rule closed a class of request-smuggling attacks).
- **`OWS` around the value is stripped.** Leading and trailing spaces/tabs are not part of the value. `X-Foo:   bar   ` carries the value `bar`. You must trim, and so does everyone else — do not encode meaning in surrounding whitespace.
- **The value is a byte string, not a typed thing.** HTTP itself has almost no idea what a field value "means." `field-value` is just visible characters and internal spaces. All structure (dates, lists, q-values, numbers) is imposed by *per-header* grammar layered on top, or — for modern headers — by [Structured Field Values](./Structured-Field-Values.md).
- **`obs-text`** (bytes 0x80–0xFF) is technically permitted in values for backward compatibility but is deprecated; treat non-ASCII in header values as suspect. Header values are conceptually US-ASCII; non-ASCII data belongs in an encoded form (e.g. RFC 8187 `filename*=UTF-8''…`).

In HTTP/2 (RFC 9113) and HTTP/3 (RFC 9114) there is no textual field line at all — fields are compressed name/value pairs (HPACK/QPACK) and the field name **must be lowercase**. The *semantics* in RFC 9110 are identical across versions; only the framing differs. This is why you should never reason about header casing on the wire (see [Case Sensitivity and Ordering](./Case-Sensitivity-and-Ordering.md)).

### token vs quoted-string

Two building blocks appear everywhere in per-header grammars:

```abnf
token          = 1*tchar
tchar          = "!" / "#" / "$" / "%" / "&" / "'" / "*"
               / "+" / "-" / "." / "^" / "_" / "`" / "|" / "~"
               / DIGIT / ALPHA

quoted-string  = DQUOTE *( qdtext / quoted-pair ) DQUOTE
qdtext         = HTAB / SP / %x21 / %x23-5B / %x5D-7E / obs-text
quoted-pair    = "\" ( HTAB / SP / VCHAR / obs-text )
```

- A **token** is a bare word with no spaces and no "separator" characters. Media types (`text/html`), methods, cache directives (`no-cache`), and encodings (`gzip`) are tokens.
- A **quoted-string** is how you carry a value that contains spaces, commas, or other separators — anything that would otherwise break list/parameter parsing. `ETag: "abc,123"` uses quoting so the comma is *inside* the value and not a list delimiter. Inside quotes, `\"` and `\\` are escapes (`quoted-pair`).

The practical rule: **if a value can contain a comma, semicolon, or space, it must be a quoted-string, and your parser must not split on delimiters that appear inside quotes.** A naive `value.split(',')` is the single most common HTTP-parsing bug. `foo="a,b", bar` is *two* list members, not three.

## The `#rule` — comma-separated lists

RFC 9110 §5.6.1 defines an ABNF extension used throughout the specs: the `#` operator, meaning a comma-separated list.

```abnf
1#element   ; one or more elements, comma-separated
#element    ; zero or more
2#5element  ; between 2 and 5 elements
```

Expanded, `1#element` is roughly `element *( OWS "," OWS element )`, but the real grammar is deliberately lenient about **empty list elements**: `foo,,bar` and `,foo , ,bar,` are all legal and mean the two-element list `[foo, bar]`. Empty elements are simply ignored. A correct list parser therefore:

1. Splits on commas **that are not inside a quoted-string**.
2. Trims OWS from each element.
3. Drops empty elements.

This leniency exists so that intermediaries can concatenate list-valued fields safely (see next section) without worrying about stray commas.

### Parameters and the `;key=value` form

Many list elements carry parameters, appended with semicolons:

```abnf
parameters      = *( OWS ";" OWS [ parameter ] )
parameter       = parameter-name "=" parameter-value
parameter-name  = token
parameter-value = token / quoted-string
```

So a media-range in `Accept` looks like `type/subtype ; param=value ; q=0.8`. The comma separates *list members*; the semicolon separates a member from *its parameters*. Two different delimiters, two different levels. `Content-Type: text/html; charset=utf-8` is a single value with one parameter — there is no list here, so a comma would be wrong.

### q-values (quality values)

The `q` parameter is HTTP's weighting mechanism for content negotiation (`Accept`, `Accept-Language`, `Accept-Encoding`, `TE`). Its grammar (RFC 9110 §12.4.2):

```abnf
weight = OWS ";" OWS "q=" qvalue
qvalue = ( "0" [ "." *3DIGIT ] )   ; 0.000 – 0.999... and 0
       / ( "1" [ "." *3("0") ] )   ; 1 or 1.000
```

Key facts engineers get wrong:

- **q ranges 0 to 1**, with at most three decimal places. `q=0.5` means "half as preferred." `q=0` means **"not acceptable at all"** — it is an explicit exclusion, not merely low priority. `Accept-Encoding: gzip, *;q=0` says "gzip is fine, refuse everything else including identity."
- **A missing q defaults to `q=1`.** So `Accept: text/html, application/xml;q=0.9` prefers HTML (implicit 1.0) over XML (0.9).
- **The `q` parameter is a delimiter between "media-range parameters" and "extension parameters"** in `Accept`. Parameters *before* `q` modify the media range and are matched against the resource; parameters *after* `q` are accept-extensions. In practice almost everyone treats `q` as the last thing that matters and ignores accept-ext.
- Selection is highest-q-wins, with specificity as the tie-breaker (`text/html` beats `text/*` beats `*/*` at equal q). The server chooses; the client only expresses preference.

## Repeated fields, combining, and why folding is dead

RFC 9110 §5.2 lays down a rule that has deep consequences:

> A sender MUST NOT generate multiple field lines with the same name in a message unless either the entire field value for that field is defined as a comma-separated list \[i.e., #(values)] **or** the field is a well-known exception (notably `Set-Cookie`).

And §5.3: a recipient MAY combine multiple field lines with the same name **into one**, by joining the values in order with a comma, and this MUST NOT change the message semantics. That is *why* the `#rule` lists exist: because list-valued headers are defined so that

```http
Cache-Control: no-cache
Cache-Control: no-store, max-age=0
```

is semantically identical to

```http
Cache-Control: no-cache, no-store, max-age=0
```

Node.js relies on exactly this. `http.IncomingMessage.headers` gives you a single string per header with duplicates joined by `, ` — *except* for a hard-coded set (`set-cookie` becomes an array, and a few others like `age`, `authorization`, `content-length` keep only the first because duplicates are illegal/nonsensical for them). If you need the un-joined lines, use `req.rawHeaders` (a flat `[name, value, name, value, …]` array) or `req.headersDistinct` (Node 18.3+, always arrays).

### Obsolete line folding

Historically, a field value could be continued onto the next line if that line started with whitespace ("obs-fold"):

```http
X-Long: this value is
  continued here
```

RFC 9112 §5.2 **deprecates and effectively forbids** this. A server that receives obs-fold in a request MUST either reject it with 400 or replace each fold with a space before processing — because folding was a rich source of request-smuggling and header-injection ambiguity. **Do not emit folded headers. Ever.** If a value is too long, that is a design problem (see [Header Size Limits](./Header-Size-Limits.md)), not a formatting one.

## The `Set-Cookie` exception

`Set-Cookie` breaks the combining rule and deserves its own callout because mishandling it is a top-tier bug.

The problem: a cookie's `Expires` attribute uses an HTTP-date, and HTTP-dates *contain commas* (`Wed, 09 Jun 2021 10:18:14 GMT`). If you combined multiple `Set-Cookie` lines with commas per the general rule, you could never unambiguously split them back apart. So RFC 9110 §5.3 and RFC 6265 (cookies) declare `Set-Cookie` a **special case that must NOT be combined** — each cookie gets its own field line, and recipients must treat them as separate.

Consequences you must respect:

- On the server, set multiple cookies as **separate** `Set-Cookie` lines. In Node, pass an **array** to `res.setHeader('Set-Cookie', [...])`; do not concatenate.
- When reading, `req.headers['set-cookie']` is the one header Node always exposes as an **array** rather than a joined string.
- The Fetch API historically hid `Set-Cookie` from JS entirely; the newer `Headers.getSetCookie()` returns the array where supported, but in browsers `Set-Cookie` is not exposed to page JS at all for security.

```http
HTTP/1.1 200 OK
Set-Cookie: session=abc123; Path=/; HttpOnly; Secure; SameSite=Lax
Set-Cookie: theme=dark; Path=/; Max-Age=31536000
```

See [Set-Cookie](../08-Cookies/Set-Cookie.md) for full attribute semantics.

## Date formats: emit IMF-fixdate, tolerate the rest

HTTP dates appear in `Date`, `Expires`, `Last-Modified`, `If-Modified-Since`, `Retry-After`, cookie `Expires`, etc. RFC 9110 §5.6.7 defines the `HTTP-date` production with three formats — but the rule is asymmetric:

```abnf
HTTP-date    = IMF-fixdate / obs-date
IMF-fixdate  = day-name "," SP date1 SP time-of-day SP GMT
             ; e.g. Sun, 06 Nov 1994 08:49:37 GMT
obs-date     = rfc850-date / asctime-date
rfc850-date  = ; Sunday, 06-Nov-94 08:49:37 GMT   (2-digit year, legacy)
asctime-date = ; Sun Nov  6 08:49:37 1994          (ANSI C, no timezone)
```

- **Always send IMF-fixdate.** Fixed-length, four-digit year, always `GMT`, `day-name` abbreviated to three letters. It is the only format you should ever generate.
- **Parse all three on input** for robustness, because ancient clients/servers still emit the obsolete forms.
- **Timezone is always GMT/UTC.** There is no offset syntax; `GMT` is literal.

In Node, `new Date().toUTCString()` produces exactly IMF-fixdate — that is the canonical way to format an HTTP date, and it is what frameworks use internally:

```js
new Date().toUTCString(); // "Tue, 07 Jul 2026 12:00:00 GMT"  ✅ IMF-fixdate
```

Do **not** use `toISOString()` (that is `2026-07-07T12:00:00.000Z` — an ISO 8601 date, wrong for HTTP headers) and do not hand-roll date strings with locale-dependent methods.

## Parsing a q-value list correctly in Node

Here is a production-grade parser for an `Accept`-style q-value list. It handles quoted-strings (so commas inside quotes don't split), OWS, empty elements, missing-q-defaults-to-1, `q=0` exclusion, and stable sorting. Every non-obvious line is annotated.

```js
/**
 * Parse an HTTP list-of-preferences header value (Accept, Accept-Language,
 * Accept-Encoding, TE) into a q-sorted array of { value, q, params }.
 */
function parseQualityList(headerValue) {
  if (!headerValue) return [];                    // absent header → empty list, caller decides default

  const members = splitOutsideQuotes(headerValue, ','); // list split — comma is the #rule delimiter
  const parsed = [];

  members.forEach((raw, index) => {
    const member = raw.trim();                    // strip OWS around each element (RFC 9112 §5)
    if (member === '') return;                    // empty list elements are legal and ignored (#rule)

    const parts = splitOutsideQuotes(member, ';'); // ';' separates the value from its parameters
    const value = parts[0].trim();                // the media-range / language-range / coding token
    if (value === '') return;                     // e.g. "; q=0.5" with no value is meaningless

    let q = 1;                                     // RFC 9110 §12.4.2: absent q defaults to 1
    const params = {};

    for (let i = 1; i < parts.length; i++) {       // remaining parts are ";key=value" parameters
      const p = parts[i].trim();
      const eq = p.indexOf('=');
      const key = (eq === -1 ? p : p.slice(0, eq)).trim().toLowerCase();
      let val = eq === -1 ? '' : p.slice(eq + 1).trim();
      if (val.startsWith('"') && val.endsWith('"')) {
        val = val.slice(1, -1).replace(/\\(.)/g, '$1'); // unescape a quoted-string parameter value
      }
      if (key === 'q') {
        const n = Number.parseFloat(val);          // qvalue is 0..1 with ≤3 decimals
        q = Number.isFinite(n) ? Math.min(Math.max(n, 0), 1) : 1; // clamp; malformed → treat as 1
      } else {
        params[key] = val;                         // preserve media-range params (e.g. charset)
      }
    }

    parsed.push({ value, q, params, index });      // keep original index for stable tie-breaking
  });

  return parsed
    .filter((m) => m.q > 0)                        // q=0 means "not acceptable" → exclude entirely
    .sort((a, b) => b.q - a.q || a.index - b.index) // highest q first; stable on ties (spec order)
    .map(({ value, q, params }) => ({ value, q, params }));
}

/**
 * Split `s` on `delimiter`, but ignore delimiters that fall inside a
 * double-quoted-string (honoring backslash escapes). This is the piece a
 * naive String.split() gets wrong.
 */
function splitOutsideQuotes(s, delimiter) {
  const out = [];
  let buf = '';
  let inQuotes = false;
  for (let i = 0; i < s.length; i++) {
    const ch = s[i];
    if (inQuotes && ch === '\\') {                 // escaped char inside quotes: keep both bytes
      buf += ch + (s[i + 1] ?? '');
      i++;
      continue;
    }
    if (ch === '"') { inQuotes = !inQuotes; buf += ch; continue; }
    if (ch === delimiter && !inQuotes) { out.push(buf); buf = ''; continue; }
    buf += ch;
  }
  out.push(buf);
  return out;
}
```

Exercising it:

```js
parseQualityList('text/html, application/xml;q=0.9, */*;q=0.8');
// → [ {value:'text/html', q:1, params:{}},
//     {value:'application/xml', q:0.9, params:{}},
//     {value:'*/*', q:0.8, params:{}} ]

parseQualityList('gzip, br;q=1.0, *;q=0');
// → [ {value:'gzip', q:1, ...}, {value:'br', q:1, ...} ]   // identity/* excluded by q=0

parseQualityList('application/json;q=0.5, text/plain;foo="a,b"');
// → text/plain (q=1) ranks first; the "a,b" comma did NOT split the list
```

In real code you would usually reach for a battle-tested library — `negotiator` (used by Express's `req.accepts()`), `accepts`, or `content-type` for `Content-Type` parsing — rather than shipping your own. But you should understand exactly what they are doing, because when negotiation goes wrong (wrong variant served, `Vary` misconfigured, a proxy caching the wrong encoding), the fix lives in this grammar. See [Content Negotiation Overview](../11-Content-Negotiation/Content-Negotiation-Overview.md).

## A few grammar gotchas worth memorizing

- **Field names are tokens and case-insensitive; values are not tokens and their case-sensitivity is per-header.** `Content-Type` value tokens are case-insensitive (`Text/HTML` == `text/html`), but an `ETag` or a cookie value is case-*sensitive* opaque data. Never blanket-lowercase a value. See [Case Sensitivity and Ordering](./Case-Sensitivity-and-Ordering.md).
- **CR and LF are illegal inside a value.** This is the entire basis of header-injection defense: if user input containing `\r\n` reaches a header value unfiltered, an attacker can inject new headers or split the response (CRLF injection / response splitting). Node's `http` module throws `ERR_INVALID_CHAR` if you try to set a header containing CR/LF — do not defeat that.
- **Duplicate singleton headers are an attack surface.** For headers defined as a single value (`Content-Length`, `Host`, `Content-Type`), two conflicting copies is exactly the ambiguity request-smuggling exploits. Reject or normalize; don't silently pick one.
- **Whitespace is not free.** Leading/trailing OWS is stripped, but *internal* spaces are significant and preserved. `User-Agent` values legitimately contain spaces; that is why `User-Agent` is not a list.

## Mental Model

Treat every header value as a **little language with three nested levels of punctuation**, and always parse outermost-first:

1. **Commas** separate list members (`#rule`). But only commas *outside quotes* — quotes are a force field.
2. **Semicolons** hang parameters off a member (`;key=value`), with `q=` acting as the priority dial.
3. **Double quotes** wrap any value that would otherwise contain a delimiter, with backslash as the escape hatch.

If you internalize "split on commas outside quotes, then split each piece on semicolons, then trim OWS, then interpret `q`," you can parse the vast majority of classic HTTP headers correctly by hand — and you'll immediately see why `value.split(',')` is a bug, why `Set-Cookie` had to opt out (its dates contain commas), and why the modern web is migrating to [Structured Field Values](./Structured-Field-Values.md) to make all of this a solved, shared problem instead of a per-header re-implementation.
