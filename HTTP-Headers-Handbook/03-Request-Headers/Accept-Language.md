# Accept-Language

## Quick Summary

`Accept-Language` is a **request** header by which the client tells the server **which natural languages the user prefers**, in ranked order with quality weights — e.g. `Accept-Language: fr-FR, fr;q=0.9, en;q=0.5`. It is the client half of **language content negotiation**: the browser sends the user's language preferences (derived from OS/browser settings), the server picks the best-matching available language, returns that representation, and labels it with [`Content-Language`](../04-Response-Headers/Content-Language.md). It uses **BCP 47** language tags with **q-values** (0.0–1.0) expressing relative preference. It's the standards-based way to serve the right language *automatically* on first visit — before any cookie or user setting exists — and it drives internationalized sites, localized error messages, and locale-aware formatting. Because the response depends on it, it must be paired with [`Vary: Accept-Language`](../06-Caching-Headers/Vary.md) so caches don't cross-serve languages. It's a *preference hint*, not a command — servers should have a sensible default and often let an explicit user choice (cookie/profile) override the header.

## What problem does this header solve?

When a user from France first visits your global site, how does the server know to show French — before they've logged in, set a preference, or clicked a language switcher? The information exists: the user's browser/OS is configured for French. `Accept-Language` is how that configured preference travels to the server automatically on the *very first* request, so you can serve French immediately instead of defaulting everyone to English and making them hunt for a switcher.

More precisely, it solves **ranked, weighted language negotiation**: users often understand multiple languages with different fluency ("French best, then English, then a little German"), and `Accept-Language` expresses that as an ordered list with q-values. The server matches the user's ranked preferences against the languages it actually offers and picks the best available. This enables:
- **Zero-config localization on first visit** (no cookie needed).
- **Graceful fallback** ("we don't have French, but you also accept English → serve English").
- **Locale-aware formatting** (dates, numbers, currency) matching the user's region.
- **Localized API responses** (error messages, content) for API clients that forward the header.

## Why was it introduced?

`Accept-Language` was introduced with HTTP/1.1 (RFC 2068, 1997; RFC 2616, 1999) as part of **proactive content negotiation** — the `Accept-*` family that lets one URL serve multiple representations chosen by client preferences. It's specified today in **RFC 9110 §12.5.4 (2022)**, and its values use **BCP 47** language tags (RFC 5646) — the same tags used by [`Content-Language`](../04-Response-Headers/Content-Language.md) and the HTML `lang` attribute — with the **q-value** weighting mechanism (RFC 9110 §12.4.2) shared across the `Accept-*` headers. It was designed so the *web platform itself* could route users to the right language without every site inventing its own detection scheme, and so servers could make principled fallback decisions from a ranked list rather than a single value. Over time, best practice evolved to treat it as a *default/first-visit* signal that an explicit user choice should be able to override (and to be mindful of its privacy/fingerprinting surface).

## How does it work?

The browser sends `Accept-Language` with the user's languages, each optionally weighted by `q` (default `q=1`); higher `q` = more preferred. The server parses the list, matches it against its supported languages, and returns the best match (with [`Content-Language`](../04-Response-Headers/Content-Language.md) + [`Vary: Accept-Language`](../06-Caching-Headers/Vary.md)).

```mermaid
sequenceDiagram
    participant B as Browser (OS/user lang settings)
    participant S as Server (supports en, fr, de)
    B->>S: GET /welcome<br/>Accept-Language: fr-FR, fr;q=0.9, en;q=0.5
    Note over S: Rank: fr-FR(1) > fr(0.9) > en(0.5)<br/>Best available match = fr
    S-->>B: 200 OK<br/>Content-Language: fr<br/>Vary: Accept-Language<br/>(French content)
```

- **Browser behavior:** Sends `Accept-Language` automatically based on the user's browser/OS language settings, as a ranked, weighted list. `navigator.language`/`navigator.languages` expose the same to JS. Users can configure/override these in browser settings.
- **Server behavior:** Parses and matches against supported languages (best-match with fallback), returns the chosen language, sets [`Content-Language`](../04-Response-Headers/Content-Language.md) and [`Vary: Accept-Language`](../06-Caching-Headers/Vary.md). Often lets an explicit user choice (cookie/profile) take precedence over the header.
- **Proxy behavior:** Forwards it; shared proxies must honor `Vary: Accept-Language` to keep variants separate.
- **CDN behavior:** Can route/serve per-language variants keyed on `Accept-Language` (often normalized to the supported set at the edge).
- **Reverse proxy behavior:** Nginx can pick a language via `map $http_accept_language` and route accordingly, or pass it to the app.

## HTTP Request Example

A ranked, weighted preference list:

```http
GET /welcome HTTP/1.1
Host: www.example.com
Accept-Language: fr-FR, fr;q=0.9, en-US;q=0.8, en;q=0.7, *;q=0.1
```

Read as: French (France) most preferred, then generic French, then US English, then generic English, then anything else as a last resort.

A single-language client:

```http
GET /api/errors HTTP/1.1
Host: api.example.com
Accept-Language: ja
```

## HTTP Response Example

The negotiated French response:

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Language: fr
Vary: Accept-Language
Cache-Control: public, max-age=300

<!doctype html><html lang="fr">...
```

If no acceptable language is available, servers typically serve a default (rather than `406`), labeled accordingly:

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Content-Language: en
Vary: Accept-Language
```

## Express.js Example

```js
const express = require('express');
const app = express();

const SUPPORTED = ['en', 'fr', 'de', 'ja'];
const DEFAULT_LANG = 'en';

app.get('/welcome', (req, res) => {
  // 1) An explicit user choice (cookie/profile) should OVERRIDE the header.
  const chosen = req.cookies?.lang && SUPPORTED.includes(req.cookies.lang)
    ? req.cookies.lang
    // 2) Otherwise negotiate from Accept-Language. Express parses q-values and
    //    matches against our supported set, returning the best fit or false.
    : (req.acceptsLanguages(SUPPORTED) || DEFAULT_LANG);

  res.set('Content-Language', chosen);  // label what we chose
  res.vary('Accept-Language');          // caches must key on the negotiated dimension
  res.set('Cache-Control', 'public, max-age=300');
  res.type('html').send(renderWelcome(chosen)); // body + <html lang> should match
});

// 3) Localized API error messages negotiated from the header.
app.use((err, req, res, next) => {
  const lang = req.acceptsLanguages(SUPPORTED) || DEFAULT_LANG;
  res.set('Content-Language', lang).vary('Accept-Language');
  res.status(err.status || 500).json({ error: localizeError(err.code, lang) });
});

// 4) i18n middleware (e.g. i18next) typically wires this up for you:
// const i18next = require('i18next');
// const middleware = require('i18next-http-middleware');
// i18next.use(middleware.LanguageDetector).init({ supportedLngs: SUPPORTED, fallbackLng: DEFAULT_LANG });
// app.use(middleware.handle(i18next)); // detects from Accept-Language (and cookie/query).

app.listen(3000);
```

Why each piece matters: the **cookie-overrides-header** precedence (step 1) is the key UX principle — `Accept-Language` is the *first-visit default*, but once a user explicitly picks a language, honor that choice on subsequent visits rather than re-guessing from the header. `req.acceptsLanguages(SUPPORTED)` (step 2) does the q-value parsing and best-match against *your* languages, returning the top supported one (or `false` → fall back to default; note best practice is to serve a default rather than `406 Not Acceptable`, which is hostile UX). `res.vary('Accept-Language')` prevents a shared cache from serving French to an English user. Step 4 notes that real apps usually delegate all this to an i18n library that detects from header + cookie + query in one place.

## Node.js Example

Raw `http` with manual q-value parsing:

```js
const http = require('http');

const SUPPORTED = ['en', 'fr', 'de'];
const DEFAULT = 'en';

function negotiate(acceptLanguage) {
  // Parse "fr-FR,fr;q=0.9,en;q=0.5" → ranked base tags.
  const ranked = (acceptLanguage || '')
    .split(',')
    .map(part => {
      const [tag, q] = part.trim().split(';q=');
      return { tag: tag.toLowerCase().split('-')[0], q: q === undefined ? 1 : parseFloat(q) };
    })
    .filter(r => !Number.isNaN(r.q))
    .sort((a, b) => b.q - a.q);
  return (ranked.find(r => SUPPORTED.includes(r.tag)) || { tag: DEFAULT }).tag;
}

http.createServer((req, res) => {
  const lang = negotiate(req.headers['accept-language']);
  res.setHeader('Content-Language', lang);
  res.setHeader('Vary', 'Accept-Language');
  res.setHeader('Content-Type', 'text/html; charset=utf-8');
  res.end(`<!doctype html><html lang="${lang}"><body>${greet(lang)}</body></html>`);
}).listen(3000);
```

The contract: parse the ranked list, match against supported languages (respecting q-values and stripping region subtags for fallback), default sensibly, and set `Content-Language` + `Vary`.

## React Example

React apps interact with language preference on the client via `navigator.languages` and by *sending* `Accept-Language` (implicitly) or a chosen locale:

1. **Reading the browser's preference in the client.** `navigator.languages` mirrors `Accept-Language` — use it to pick an initial locale for a client-rendered app:

```jsx
function useInitialLocale(supported, fallback = 'en') {
  return React.useMemo(() => {
    // navigator.languages is the client-side view of Accept-Language.
    for (const l of navigator.languages || [navigator.language]) {
      const base = l.toLowerCase().split('-')[0];
      if (supported.includes(base)) return base;
    }
    return fallback;
  }, [supported, fallback]);
}
```

2. **Explicit override + sending it to APIs.** Once a user picks a language, store it (cookie/localStorage) and send it on API calls so the backend returns localized data:

```jsx
const api = axios.create();
api.interceptors.request.use((config) => {
  const lang = localStorage.getItem('lang') || navigator.language;
  config.headers['Accept-Language'] = lang;  // negotiate localized API responses
  return config;
});
```

3. **SSR/Next.js.** Server-rendered React reads `Accept-Language` (via the request) to render the right locale on first paint, then respects a cookie/route override — the standard i18n-routing pattern. Set `<html lang>` and `Content-Language` to match.

Key point: `Accept-Language` is a fine *default*, but persist and honor the user's explicit choice; don't re-detect from the header on every visit once they've chosen.

## Browser Lifecycle

1. The browser derives `Accept-Language` from the user's OS/browser language settings (a ranked list) and sends it on requests automatically.
2. The server negotiates and returns the best-matching language with [`Content-Language`](../04-Response-Headers/Content-Language.md) + [`Vary: Accept-Language`](../06-Caching-Headers/Vary.md).
3. The browser (and shared caches) key cached copies on `Accept-Language` per `Vary`, so different-language users get correct variants.
4. `navigator.language`/`navigator.languages` expose the same preference to page JS.
5. Users can change their language settings; subsequent requests reflect the new list.
6. An app-level explicit choice (cookie) typically overrides the header on the server.

## Production Use Cases

- **First-visit localization:** serve the right language automatically before any user setting exists.
- **Localized API responses:** error messages, content, and validation messages negotiated per client.
- **Locale-aware formatting:** choose date/number/currency formats matching the user's region.
- **Multilingual SEO:** combined with `hreflang`/[`Content-Language`](../04-Response-Headers/Content-Language.md) for correct language targeting.
- **Fallback chains:** honor secondary preferences when the primary isn't available.
- **Edge language routing:** CDN picks a locale variant from the header for faster localized delivery.

## Common Mistakes

- **Ignoring q-values / list order.** Taking only the first tag misses weighted preferences and fallbacks; parse the ranked list.
- **Not stripping region for fallback.** A client asking `fr-CA` when you only have `fr` should still get French — match base tags after exact tags.
- **Returning `406 Not Acceptable` when no match.** Hostile UX; serve a sensible default instead.
- **Omitting [`Vary: Accept-Language`](../06-Caching-Headers/Vary.md).** Shared caches cross-serve languages (English user gets French).
- **Letting the header override an explicit user choice.** Once a user picks a language, honor it; don't re-guess from the header.
- **Over-trusting it for identity/region.** It reflects preferences, not location or identity; don't gate access on it.
- **Cache fragmentation.** Keying caches on the raw high-cardinality header explodes variants — normalize to your supported set.
- **Non-BCP-47 handling.** Use standard tags and case-insensitive matching.

## Security Considerations

- **Fingerprinting/privacy.** `Accept-Language` contributes to browser fingerprinting (the specific language list + order is somewhat identifying) and can hint at a user's nationality/region — a privacy consideration. Some privacy tools normalize or strip it. Don't rely on it as a secret or identity signal.
- **Not identity or geolocation.** It's a client-set preference (spoofable); never use it for access control, pricing discrimination that violates policy, or security decisions.
- **Injection hygiene.** If you reflect the negotiated language (or raw header) into responses/logs, validate against your supported BCP 47 set to avoid header/log injection.
- **Cache correctness (mild security angle).** Failing to `Vary` can serve the wrong-language variant, which for locale-gated content could disclose unintended regional content.

## Performance Considerations

- **Cache fragmentation by language.** `Vary: Accept-Language` creates one cache entry per language variant; **normalize** the raw header to your small supported set before keying so you store a few variants, not thousands.
- **Negligible header cost;** the impact is cache-key design.
- **Edge language routing** can reduce latency by serving the right variant from the CDN.
- **Negotiation is cheap** (a small parse/match); do it once per request.
- **First-visit correctness** avoids a redirect/reload to switch languages, improving perceived performance.

## Reverse Proxy Considerations

Nginx picking a language via `map` and normalizing for cache keys:

```nginx
http {
  # Map the raw Accept-Language to a normalized supported language.
  map $http_accept_language $preferred_lang {
    default        en;
    ~*^fr          fr;
    ~*^de          de;
    ~*^ja          ja;
  }

  server {
    location / {
      proxy_pass http://app_upstream;
      proxy_set_header X-Preferred-Lang $preferred_lang;  # hand the app a normalized value
      # Normalize the cache key on the SUPPORTED language, not the raw header:
      proxy_cache_key "$scheme$host$request_uri$preferred_lang";
      add_header Vary "Accept-Language" always;
    }
  }
}
```

Key points: normalizing the raw, high-cardinality `Accept-Language` down to your supported set (`$preferred_lang`) before using it in the cache key is the crucial move — it bounds cache variants to a handful. The `map` does a simple prefix match; the app can trust the normalized `X-Preferred-Lang`.

## CDN Considerations

- **Variant caching:** CDNs cache per-language variants keyed on `Accept-Language` (via `Vary` or a normalized custom cache key). **Normalize** to your supported set to keep hit ratio high — raw `Accept-Language` is extremely high-cardinality.
- **Edge language routing:** some CDNs pick a locale at the edge (by header or geo) and serve/redirect to the matching variant.
- **Cloudflare/CloudFront/Fastly** support language-based cache keys/routing; configure normalization.
- **Consistency:** ensure the language dimension is part of the cache key wherever content varies by language.

## Cloud Deployment Considerations

- **API Gateways:** ensure `Accept-Language` is forwarded to the backend and `Vary: Accept-Language` is preserved.
- **Managed platforms (Vercel/Netlify):** i18n routing reads `Accept-Language` for locale detection/redirects; configure default + supported locales and cache keying.
- **Serverless:** negotiate in the handler; normalize before any caching.
- **Edge functions:** a common place to normalize `Accept-Language` and route to a locale.

## Debugging

- **Chrome DevTools → Network:** inspect the request's `Accept-Language` (set via browser language settings) and the response's [`Content-Language`](../04-Response-Headers/Content-Language.md)/`Vary`.
- **curl:** `curl -sD - -o /dev/null -H 'Accept-Language: fr' https://host/welcome | grep -i 'content-language\|vary'` → confirm French + `Vary`. Try `de`, `ja`, and an unsupported tag to test fallback.
- **Change browser language** and reload to see negotiation in action.
- **Postman / Bruno:** send different `Accept-Language` values and assert the returned language.
- **Cache cross-serve test:** request `fr` then `en` through the CDN and confirm no cross-serving.
- **`navigator.languages`:** log it in the client to see what the browser would send.

## Best Practices

- [ ] Parse the **ranked, q-weighted** list and match against your **supported** set (exact tag, then base-tag fallback).
- [ ] Serve a sensible **default** when no match; avoid `406`.
- [ ] Let an **explicit user choice** (cookie/profile) override the header, and persist it.
- [ ] Set [`Content-Language`](../04-Response-Headers/Content-Language.md) + [`Vary: Accept-Language`](../06-Caching-Headers/Vary.md) on negotiated responses.
- [ ] **Normalize** the raw header to your supported set before using it in cache keys.
- [ ] Keep `Content-Language`, `<html lang>`, and content consistent.
- [ ] Don't use it for identity/geolocation/security; be mindful of fingerprinting/privacy.
- [ ] For APIs, forward `Accept-Language` and localize responses accordingly.

## Related Headers

- [Content-Language](../04-Response-Headers/Content-Language.md) — the response header naming the chosen language.
- [Vary](../06-Caching-Headers/Vary.md) — `Vary: Accept-Language` keeps language variants separate in caches.
- [Accept](./Accept.md) — media-type negotiation sibling (same q-value mechanism).
- [Accept-Encoding](../10-Compression/Accept-Encoding.md) — compression negotiation sibling.
- [Accept-Charset](./Accept-Charset.md) — the (largely obsolete) charset-negotiation sibling.
- [Content-Location](../04-Response-Headers/Content-Location.md) — can point to the specific-language variant URL.
- [Content Negotiation Overview](../11-Content-Negotiation/Content-Negotiation-Overview.md) — the framing chapter.

## Decision Tree

```mermaid
flowchart TD
    A[Request arrives] --> B{Explicit user language choice<br/>(cookie/profile)?}
    B -- Yes --> C[Use it (override header)]
    B -- No --> D[Parse Accept-Language ranked list]
    D --> E{Best match in supported set?}
    E -- Yes (exact or base tag) --> F[Serve that language]
    E -- No --> G[Serve default language]
    F --> H[Set Content-Language + Vary: Accept-Language]
    G --> H
    C --> H
    H --> I[Normalize to supported set for cache key]
```

## Mental Model

Think of `Accept-Language` as a **traveler handing a hotel concierge a small card listing the languages they speak, best first**: "French (fluent), then English (good), a little German." The concierge ([the server](../04-Response-Headers/Content-Language.md)) glances at what languages the *staff* actually speak and picks the best overlap — French if available, otherwise English, otherwise the house default — and greets the guest accordingly, noting on the guest file which language was used (`Content-Language`). Two refinements make it work smoothly: the card is *ranked with confidence levels* (q-values), so the concierge doesn't just grab the first entry but weighs preferences and can gracefully fall back; and it's the guest's *opening* signal — the moment the guest *explicitly says* "actually, please use English from now on" (a cookie/preference), the concierge honors that standing instruction and stops re-reading the card each visit. Crucially, the card states *preferences, not passport nationality* — a French-speaker might be from anywhere — so the concierge never uses it to decide *who the guest is* or *what they're allowed to do*, only *how to speak to them*.
