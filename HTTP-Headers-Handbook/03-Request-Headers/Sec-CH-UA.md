# Sec-CH-UA (Client Hints)

## Quick Summary

`Sec-CH-UA` and its siblings are **request** headers that make up **User-Agent Client Hints (UA-CH)** — the modern, structured, privacy-respecting replacement for the sprawling, unreliable [`User-Agent`](./User-Agent.md) string. Instead of one opaque free-text UA string that everyone parses with fragile regexes, UA-CH breaks the same information into discrete, opt-in headers: `Sec-CH-UA` (brand + significant version list), `Sec-CH-UA-Mobile` (`?0`/`?1` boolean), `Sec-CH-UA-Platform` (`"Windows"`, `"macOS"`, etc.), plus **high-entropy** hints the server must explicitly request — `Sec-CH-UA-Platform-Version`, `Sec-CH-UA-Arch`, `Sec-CH-UA-Model`, `Sec-CH-UA-Full-Version-List`, `Sec-CH-UA-Bitness`. The design is **default-low-entropy** (browsers send only a few coarse hints unless the server asks for more via [`Accept-CH`](../04-Response-Headers/Content-Type.md)), which reduces passive fingerprinting. UA-CH uses [Structured Field Values (RFC 8941)](../02-Core-Concepts/Structured-Field-Values.md), includes deliberate **"GREASE"** entries to prevent brittle parsing, and is part of the broader effort to **freeze and eventually reduce** the legacy `User-Agent` string. For production work, UA-CH is how you should do device/browser adaptation going forward — with the crucial caveats that hints are **spoofable**, **opt-in**, and **Chromium-centric** (Firefox/Safari have limited or no support).

## What problem does this header solve?

The legacy [`User-Agent`](./User-Agent.md) string is a decades-long disaster: a single opaque line like `Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0 Safari/537.36` that is (a) **unparseable reliably** (everyone pretends to be everyone else for compatibility — "Mozilla", "like Gecko", "Safari" all appear in a Chrome UA), (b) a **major fingerprinting vector** (it's sent to every server on every request, packed with identifying detail), and (c) **always sent in full**, whether or not the server needs any of it.

UA-CH solves all three:
- **Structured, reliable parsing:** discrete headers with defined syntax ([Structured Fields](../02-Core-Concepts/Structured-Field-Values.md)) instead of regex-guessing an opaque string.
- **Reduced passive fingerprinting:** only *low-entropy* hints (browser brand, mobile-ness, coarse platform) are sent by default; **high-entropy** hints (exact OS version, CPU arch, device model) are sent **only when the server explicitly requests them** via [`Accept-CH`](../04-Response-Headers/Content-Type.md) — so the default footprint shrinks.
- **Opt-in, need-to-know delivery:** servers declare exactly which hints they need; browsers can decide whether to honor requests (and privacy settings can restrict them).

It lets you do legitimate adaptation (serve the right image density, offer the right OS-specific download, detect mobile) **without** every site passively harvesting a full device fingerprint from everyone.

## Why was it introduced?

UA-CH is part of the **Client Hints** framework and specifically the **User-Agent Client Hints** specification (W3C/WICG, ~2020, led by Chromium), created to enable the long-planned **User-Agent Reduction/Freeze**: browsers want to stop growing (and eventually shrink) the legacy `User-Agent` string because of its fingerprinting harm and parsing fragility, but they can't just remove information sites legitimately need. UA-CH is the replacement channel — structured, opt-in, privacy-tiered. The `Sec-` prefix marks these as [forbidden headers](../02-Core-Concepts/Forbidden-and-Restricted-Headers.md) that **JavaScript cannot set** (only the browser can), guaranteeing their integrity from page scripts and signaling they're browser-controlled security-relevant headers. The framework uses [RFC 8941 Structured Fields](../02-Core-Concepts/Structured-Field-Values.md) for machine-friendliness and includes **GREASE** (deliberately random/fake brand entries) to force robust parsing and prevent the ossification that ruined the old UA string. Adoption is **Chromium-driven**; Firefox and Safari have historically been skeptical/limited, so UA-CH is not universally available.

## How does it work?

By default the browser sends a few **low-entropy** hints on requests. To get **high-entropy** hints, the server responds with [`Accept-CH`](../04-Response-Headers/Content-Type.md) listing the hints it wants; the browser then includes those on *subsequent* requests. Servers should also add [`Vary`](../06-Caching-Headers/Vary.md) / `Critical-CH` appropriately.

```mermaid
sequenceDiagram
    participant B as Browser
    participant S as Server
    B->>S: GET / <br/>Sec-CH-UA: "Chromium";v="120", "Not?A_Brand";v="24"<br/>Sec-CH-UA-Mobile: ?0<br/>Sec-CH-UA-Platform: "Windows"
    Note over S: I also need OS version + arch → request them
    S-->>B: 200 OK<br/>Accept-CH: Sec-CH-UA-Platform-Version, Sec-CH-UA-Arch<br/>Vary: Sec-CH-UA-Platform-Version
    Note over B: Remember which hints this origin wants
    B->>S: GET /next (NOW includes the high-entropy hints)<br/>Sec-CH-UA-Platform-Version: "15.0.0"<br/>Sec-CH-UA-Arch: "x86"
```

Low-entropy (sent by default): `Sec-CH-UA`, `Sec-CH-UA-Mobile`, `Sec-CH-UA-Platform`.
High-entropy (opt-in via `Accept-CH`): `Sec-CH-UA-Platform-Version`, `Sec-CH-UA-Arch`, `Sec-CH-UA-Bitness`, `Sec-CH-UA-Model`, `Sec-CH-UA-Full-Version-List`, `Sec-CH-UA-WoW64`.

- **Browser behavior:** Sends low-entropy hints by default (Chromium); adds high-entropy hints only after an [`Accept-CH`](../04-Response-Headers/Content-Type.md) opt-in (persisted per origin) or when the JS `navigator.userAgentData.getHighEntropyValues()` API is called. Includes GREASE entries in `Sec-CH-UA`. Won't let page JS set `Sec-CH-*` (forbidden). Firefox/Safari support is limited/absent.
- **Server behavior:** Requests needed hints via `Accept-CH`, uses them for adaptation, and must `Vary` on any hint that changes the response (for caching correctness). `Critical-CH` can force a retry so the first response already has the hints.
- **Proxy/CDN behavior:** Must pass hints and `Accept-CH`/`Vary` through; caching correctness depends on varying on the hints used.
- **Reverse proxy behavior:** Can request/consume hints; must include them in cache keys where they affect the response.

## HTTP Request Example

Default low-entropy hints on a first request:

```http
GET / HTTP/1.1
Host: www.example.com
Sec-CH-UA: "Chromium";v="120", "Google Chrome";v="120", "Not?A_Brand";v="24"
Sec-CH-UA-Mobile: ?0
Sec-CH-UA-Platform: "Windows"
```

After the server opted in via `Accept-CH`, a later request includes high-entropy hints:

```http
GET /download HTTP/1.1
Host: www.example.com
Sec-CH-UA: "Chromium";v="120", "Google Chrome";v="120", "Not?A_Brand";v="24"
Sec-CH-UA-Mobile: ?0
Sec-CH-UA-Platform: "Windows"
Sec-CH-UA-Platform-Version: "15.0.0"
Sec-CH-UA-Arch: "x86"
Sec-CH-UA-Bitness: "64"
```

## HTTP Response Example

The server requesting hints and varying on them:

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Accept-CH: Sec-CH-UA-Platform-Version, Sec-CH-UA-Arch, Sec-CH-UA-Model
Critical-CH: Sec-CH-UA-Platform-Version
Vary: Sec-CH-UA-Platform-Version, Sec-CH-UA-Arch
```

- `Accept-CH` — "send me these hints going forward."
- `Critical-CH` — "these are critical; if you didn't send them, retry now so I can respond correctly the first time."
- `Vary` — cache correctness for hint-dependent responses.

## Express.js Example

```js
const express = require('express');
const app = express();

// 1) Request the high-entropy hints you actually need (need-to-know only).
app.use((req, res, next) => {
  res.set('Accept-CH', 'Sec-CH-UA-Platform-Version, Sec-CH-UA-Arch, Sec-CH-UA-Bitness');
  // Critical-CH forces the browser to retry with these on the very next load,
  // so you don't lose a round-trip on first visit for critical adaptation.
  res.set('Critical-CH', 'Sec-CH-UA-Platform-Version');
  next();
});

// 2) Use low-entropy hints (always available on Chromium) for coarse decisions.
app.get('/', (req, res) => {
  const isMobile = req.headers['sec-ch-ua-mobile'] === '?1';
  const platform = (req.headers['sec-ch-ua-platform'] || '').replace(/"/g, ''); // strip quotes
  // If the response depends on these, VARY on them for cache correctness.
  res.vary('Sec-CH-UA-Mobile, Sec-CH-UA-Platform');
  res.type('html').send(renderHome({ isMobile, platform }));
});

// 3) Use high-entropy hints (only present after Accept-CH opt-in) for precise choices,
//    e.g. offering the correct OS/arch-specific download.
app.get('/download', (req, res) => {
  const platform = (req.headers['sec-ch-ua-platform'] || '').replace(/"/g, '');
  const arch = (req.headers['sec-ch-ua-arch'] || '').replace(/"/g, '');       // "x86" | "arm"
  const bitness = (req.headers['sec-ch-ua-bitness'] || '').replace(/"/g, ''); // "64"
  res.vary('Sec-CH-UA-Platform, Sec-CH-UA-Arch, Sec-CH-UA-Bitness');
  // Fall back gracefully if hints are absent (non-Chromium, privacy settings, first visit).
  const build = pickDownload(platform || detectFromUA(req.headers['user-agent']), arch, bitness);
  res.json({ download: build });
});

app.listen(3000);
```

Why each piece matters: `Accept-CH` (step 1) is the **opt-in** — high-entropy hints simply won't arrive unless you request them (and only from Chromium). `Critical-CH` avoids the "first response can't adapt because the hint wasn't sent yet" round-trip by making the browser retry. The **quote-stripping** is a real gotcha: [structured-field](../02-Core-Concepts/Structured-Field-Values.md) string values arrive *quoted* (`"Windows"`), so `=== 'Windows'` fails without stripping. **`Vary`** on the hints you use is essential (step 2/3) or a shared cache serves the mobile page to desktop users. And crucially, **graceful fallback** (step 3): hints are absent on Firefox/Safari, under privacy settings, and on first visit — so always fall back (e.g. to a coarse `User-Agent` parse or a safe default), never *require* a hint.

## Node.js Example

Raw `http`:

```js
const http = require('http');

http.createServer((req, res) => {
  // Opt in to the hints we need.
  res.setHeader('Accept-CH', 'Sec-CH-UA-Platform-Version, Sec-CH-UA-Arch');

  const mobile = req.headers['sec-ch-ua-mobile'] === '?1';
  const platform = (req.headers['sec-ch-ua-platform'] || '').replace(/"/g, '');
  const platformVersion = (req.headers['sec-ch-ua-platform-version'] || '').replace(/"/g, '');

  // Vary on anything that shapes the response.
  res.setHeader('Vary', 'Sec-CH-UA-Mobile, Sec-CH-UA-Platform');
  res.setHeader('Content-Type', 'application/json');
  res.end(JSON.stringify({
    mobile, platform, platformVersion: platformVersion || null, // null when not opted-in yet
    note: platformVersion ? 'high-entropy present' : 'request Accept-CH first / non-Chromium',
  }));
}).listen(3000);
```

The pattern: opt in via `Accept-CH`, read low-entropy hints immediately, high-entropy only after opt-in, strip quotes, `Vary`, and tolerate absence.

## React Example

React apps consume UA-CH via the **`navigator.userAgentData`** JS API (the client-side counterpart to the headers):

```jsx
function useUserAgentData() {
  const [data, setData] = React.useState({ mobile: null, platform: null, highEntropy: null });

  React.useEffect(() => {
    // navigator.userAgentData exists on Chromium; undefined on Firefox/Safari.
    const uaData = navigator.userAgentData;
    if (!uaData) {
      // Fallback: coarse detection from navigator.userAgent, or feature detection.
      setData({ mobile: /Mobi/.test(navigator.userAgent), platform: 'unknown', highEntropy: null });
      return;
    }
    // Low-entropy is available synchronously.
    const base = { mobile: uaData.mobile, platform: uaData.platform };
    // High-entropy requires an async request (and user permission model).
    uaData.getHighEntropyValues(['platformVersion', 'architecture', 'model'])
      .then(high => setData({ ...base, highEntropy: high }))
      .catch(() => setData({ ...base, highEntropy: null }));
  }, []);

  return data;
}

function DownloadButton() {
  const { platform, highEntropy } = useUserAgentData();
  // Offer the right build — but ALWAYS provide a generic fallback link.
  const build = pickBuild(platform, highEntropy?.architecture) || 'generic';
  return <a href={`/downloads/${build}`}>Download</a>;
}
```

Key points for React devs:
1. **Client-side API mirrors the headers:** `navigator.userAgentData` (low-entropy sync, high-entropy via `getHighEntropyValues()`), while the server reads the `Sec-CH-*` headers.
2. **Not universal:** `navigator.userAgentData` is Chromium-only — **always feature-detect and fall back** (to `navigator.userAgent` or generic behavior).
3. **Don't set `Sec-CH-*` in fetch:** they're forbidden headers; the browser controls them.
4. **Prefer feature detection over UA sniffing** where possible; use UA-CH for the genuinely UA-dependent cases (downloads, coarse device class), not for capability decisions.

## Browser Lifecycle

1. On a request, the browser attaches **low-entropy** hints (`Sec-CH-UA`, `-Mobile`, `-Platform`) by default (Chromium), including GREASE entries in `Sec-CH-UA`.
2. If the response includes [`Accept-CH`](../04-Response-Headers/Content-Type.md), the browser **persists** which hints this origin wants and includes them on **subsequent** requests.
3. `Critical-CH` triggers an immediate **retry** so the first meaningful response can use the hint.
4. High-entropy hints are gated: sent only after `Accept-CH`, or fetched via `navigator.userAgentData.getHighEntropyValues()` (which the browser may govern by privacy settings/permissions).
5. Page JS **cannot set** `Sec-CH-*` (forbidden headers).
6. Non-Chromium browsers may send **none** of these — code must not depend on them.

## Production Use Cases

- **Reliable device/browser detection** without regex-parsing `User-Agent` (coarse: mobile vs desktop, platform).
- **OS/arch-specific downloads:** offer the correct installer (Windows x64 vs macOS ARM) using high-entropy hints (with fallback).
- **Responsive image/asset selection** by device class (though CSS/`srcset`/media Client Hints often better).
- **Analytics/telemetry** with structured, parseable browser/platform data.
- **Progressive migration off `User-Agent`** ahead of UA-string reduction.
- **Compatibility shims** targeting specific browser versions via `Sec-CH-UA-Full-Version-List`.

## Common Mistakes

- **Forgetting to strip quotes.** Structured-field strings are quoted (`"Windows"`); comparisons fail without stripping.
- **Not opting in via [`Accept-CH`](../04-Response-Headers/Content-Type.md).** High-entropy hints never arrive unless requested.
- **Depending on hints being present.** Absent on Firefox/Safari, under privacy settings, and on first visit — always fall back.
- **Missing [`Vary`](../06-Caching-Headers/Vary.md).** Hint-dependent responses without `Vary` get cross-served by shared caches (mobile page to desktop).
- **Over-requesting high-entropy hints.** Requesting arch/model/version you don't need re-introduces fingerprinting; request need-to-know only.
- **Trying to set `Sec-CH-*` from JS.** Forbidden/ignored; use `navigator.userAgentData`.
- **Treating hints as trustworthy.** They're spoofable (browser extensions, non-conforming clients); don't use for security decisions.
- **Cache fragmentation.** Varying on high-cardinality hints (exact version/model) can shred the cache — normalize or vary only on coarse hints.

## Security Considerations

- **Spoofable — not a security signal.** Like [`User-Agent`](./User-Agent.md), UA-CH values can be forged (extensions, privacy tools, custom clients). **Never** use them for authentication, authorization, or security gating.
- **Fingerprinting trade-off.** UA-CH *reduces* passive fingerprinting by defaulting to low entropy and gating high entropy — but **over-requesting** high-entropy hints via `Accept-CH` re-creates a fingerprint. Request only what you need; be aware regulators/privacy tooling watch this.
- **`Sec-` prefix integrity.** Being forbidden headers, page JS can't forge them *within a conforming browser* — but that only guarantees integrity against page scripts, not against a hostile client.
- **Privacy compliance:** high-entropy device data may be PII-adjacent; handle per your privacy policy (GDPR/CCPA).
- **Don't leak via caching:** ensure hint-varied responses don't inadvertently expose one user's variant to another (correct `Vary`/keys).

## Performance Considerations

- **Smaller default requests:** low-entropy hints are compact vs the bloated `User-Agent` string; and high-entropy hints are only sent when needed — a modest bandwidth/privacy win.
- **Extra round-trip risk:** high-entropy hints require an `Accept-CH` opt-in, so the *first* response can't use them unless you use `Critical-CH` (which forces a retry — itself a round-trip). Design adaptation to tolerate the first request lacking hints.
- **Cache-key impact:** `Vary` on hints fragments the cache; keep varied hints **coarse/low-cardinality** (mobile, platform) and avoid varying on exact versions/models unless necessary.
- **Structured parsing is cheap and reliable** vs regex UA parsing.

## Reverse Proxy Considerations

Nginx requesting hints and varying correctly:

```nginx
server {
  location / {
    proxy_pass http://app_upstream;
    add_header Accept-CH "Sec-CH-UA-Platform-Version, Sec-CH-UA-Arch" always;
    add_header Critical-CH "Sec-CH-UA-Platform-Version" always;
    # If the RESPONSE varies by a hint, include it in Vary AND the cache key.
    add_header Vary "Sec-CH-UA-Mobile, Sec-CH-UA-Platform" always;
    proxy_cache_key "$scheme$host$request_uri$http_sec_ch_ua_mobile$http_sec_ch_ua_platform";
  }
}
```

Key points: request hints via `Accept-CH`, and if the response depends on a hint, both `Vary` on it *and* include it in the cache key (`$http_sec_ch_ua_*`). Keep varied hints coarse to avoid cache explosion.

## CDN Considerations

- **Pass hints + `Accept-CH`/`Critical-CH`/`Vary` through**, and include used hints in the CDN cache key (via `Vary` or custom key config).
- **Coarse variance only:** varying on `Sec-CH-UA-Mobile`/`-Platform` is manageable; varying on exact version/model explodes variants.
- **Some CDNs offer device detection** built on UA-CH + `User-Agent`; may be simpler than DIY.
- **Cloudflare/CloudFront/Fastly** can inject `Accept-CH` and key on hints; verify configuration.
- **Fallback:** since non-Chromium clients send no hints, ensure the CDN default variant is sensible.

## Cloud Deployment Considerations

- **Managed platforms:** set `Accept-CH`/`Vary` via config/middleware; ensure edge caching keys on used hints.
- **Analytics pipelines:** prefer UA-CH structured fields over parsing `User-Agent` for browser/platform dimensions.
- **API gateways:** pass `Sec-CH-*` through if backends adapt on them.
- **Privacy governance:** document which high-entropy hints you request and why (data-minimization/compliance).

## Debugging

- **Chrome DevTools → Network → Headers:** inspect `Sec-CH-UA*` on requests and `Accept-CH`/`Critical-CH`/`Vary` on responses. The Console can run `navigator.userAgentData` and `getHighEntropyValues([...])`.
- **curl (simulate):** `curl -H 'Sec-CH-UA-Mobile: ?1' -H 'Sec-CH-UA-Platform: "Android"' https://host/` to test server handling.
- **Opt-in flow:** confirm the second request (after `Accept-CH`) includes the high-entropy hints; use `Critical-CH` to see the retry.
- **Cross-browser check:** verify graceful fallback on Firefox/Safari (no hints) and under privacy settings.
- **Cache test:** request as mobile then desktop and confirm no cross-serving (correct `Vary`).
- **Quote handling:** log raw header values to remember they're quoted.

## Best Practices

- [ ] Request **only the high-entropy hints you truly need** via [`Accept-CH`](../04-Response-Headers/Content-Type.md) (data minimization).
- [ ] Use `Critical-CH` for hints required to render the first response correctly.
- [ ] **Strip quotes** from structured-field string values before comparing.
- [ ] [`Vary`](../06-Caching-Headers/Vary.md) on any hint that shapes the response, and include it in cache keys — keep varied hints **coarse**.
- [ ] **Always fall back** (Firefox/Safari, privacy settings, first visit send no/limited hints).
- [ ] Consume client-side via `navigator.userAgentData` with **feature detection**; don't set `Sec-CH-*` from JS.
- [ ] **Never** use UA-CH for security/authorization — it's spoofable.
- [ ] Prefer **feature detection** over UA sniffing where the decision is about capabilities.

## Related Headers

- [User-Agent](./User-Agent.md) — the legacy string UA-CH replaces/augments; still the fallback for non-Chromium clients.
- [Accept-CH](../04-Response-Headers/Content-Type.md) — the response header that opts in to receiving high-entropy hints (and `Critical-CH` to force a retry).
- [Vary](../06-Caching-Headers/Vary.md) — required for cache correctness when responses depend on hints.
- [Structured Field Values](../02-Core-Concepts/Structured-Field-Values.md) — the RFC 8941 syntax UA-CH uses (quoted strings, booleans, lists).
- [Forbidden and Restricted Headers](../02-Core-Concepts/Forbidden-and-Restricted-Headers.md) — why JS can't set `Sec-CH-*`.
- [Custom and X- Headers](../02-Core-Concepts/Custom-and-X-Headers.md) — context on the `Sec-` prefix convention.
- [DNT / Sec-GPC](./DNT-Sec-GPC.md) — other privacy-related request signals.

## Decision Tree

```mermaid
flowchart TD
    A[Need device/browser info?] --> B{Is it a capability decision?}
    B -- Yes --> C[Prefer feature detection, not UA-CH]
    B -- No (which OS/arch/mobile) --> D{Low-entropy enough?<br/>(mobile / platform / brand)}
    D -- Yes --> E[Use default Sec-CH-UA / -Mobile / -Platform<br/>+ Vary on them]
    D -- No (exact version/arch/model) --> F[Request via Accept-CH (need-to-know)<br/>+ Critical-CH + Vary]
    E --> G[Always fall back (non-Chromium / privacy)]
    F --> G
    A --> H[Never use for security/auth (spoofable)]
```

## Mental Model

Think of UA-CH as replacing a **rambling, unreliable self-introduction shouted at everyone** with a **polite, itemized ID card you show only the details that are asked for.** The old [`User-Agent`](./User-Agent.md) was like a stranger who, to *every* shopkeeper, blurts a long, confusing life story full of half-truths ("I'm Mozilla, also kind of Safari, like Gecko...") — impossible to parse and oversharing with everyone. UA-CH instead hands over a **structured card**: the top line (brand, mobile-or-not, rough platform) is shown freely because it's harmless, but the sensitive details (exact OS version, CPU architecture, device model) stay **in your wallet** unless a shopkeeper specifically asks ("I need your OS version to give you the right installer" → `Accept-CH`), and even then the browser mediates whether to reveal them. This **need-to-know disclosure** is the whole point: legitimate shops get what they genuinely need for service, while casual observers can no longer vacuum up a full fingerprint from everyone who walks by. Two lasting cautions: the card is only issued by *some* browsers (Chromium), so you must handle strangers who carry no card at all (fall back); and the card is **easily forged**, so it's fine for tailoring service but never for deciding who someone *is*.
