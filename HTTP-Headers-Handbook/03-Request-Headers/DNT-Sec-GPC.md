# DNT / Sec-GPC

## Quick Summary

`DNT` ("Do Not Track") and `Sec-GPC` ("Global Privacy Control") are **request** headers that carry a user's **privacy preference** — a signal that the user does **not** want to be tracked or have their personal data sold/shared. Both are binary: `DNT: 1` and `Sec-GPC: 1` mean "the user is opting out." They look almost identical but have opposite fates: **`DNT` is effectively dead** — it was never legally binding, most sites ignored it, and browsers have largely removed it (Safari dropped it, Chrome/Firefox deprecated/removed the UI) because, ironically, sending it added fingerprinting entropy without producing compliance. **`Sec-GPC` is very much alive and legally significant** — it is explicitly recognized under **California's CCPA/CPRA** (and treated as a valid opt-out under some other privacy laws) as a legally-binding "do not sell or share my personal information" signal that businesses subject to those laws **must honor**. The critical practical difference: ignoring `DNT` had no consequences; **ignoring `Sec-GPC` can be a legal violation** for covered businesses. Both are set by the browser (JS can read `DNT` via `navigator.doNotTrack` and `Sec-GPC` via `navigator.globalPrivacyControl`), and both are advisory *to the point of enforcement by law* rather than by the browser.

## What problem does this header solve?

Users increasingly want to **opt out of tracking and data selling** without configuring it site-by-site (which is impractical across the whole web). The problem is delivering a single, machine-readable "don't track / don't sell my data" preference to every site automatically. Both headers attempt this — the browser sends the signal on every request, so a user sets it *once* and it applies everywhere.

The difference is *what makes a site actually comply*:
- **`DNT`** tried to solve it purely through **voluntary industry cooperation** — there was no law requiring anyone to honor it, so almost no one did, and it failed.
- **`Sec-GPC`** solves it by being **backed by law**. Under CCPA/CPRA (California) and similar regimes, a business must provide a way to opt out of the "sale" or "sharing" of personal information, and regulators/legislation have recognized GPC as a valid, legally-binding opt-out mechanism. So `Sec-GPC: 1` is not a polite request a site *may* ignore — for a covered business, it's an opt-out the site is **legally obligated to act on** (e.g. disabling ad-tech data sharing, not selling data to third parties).

So the real problem `Sec-GPC` solves is **automated, enforceable, universal privacy opt-out** — turning a per-site legal right into a one-setting browser signal.

## Why was it introduced?

**`DNT`** emerged around 2009–2011 (proposed by researchers, then a W3C Tracking Preference Expression effort) as a voluntary standard: browsers would send `DNT: 1` and sites would *choose* to respect it. The W3C effort **stalled and was ultimately discontinued (2018)** because industry never agreed to honor it and there was no enforcement. Its legacy is a cautionary tale: a privacy signal with no legal teeth is ignored, and it *worsened* privacy slightly by adding fingerprinting entropy — which is why browsers removed it.

**`Sec-GPC`** was introduced (~2020, by a coalition including privacy orgs and browsers) specifically to **succeed where DNT failed by tying the signal to law**. Timed with **CCPA (2020) / CPRA**, GPC was designed to be a recognized "opt-out of sale/sharing" mechanism under those statutes. The California Attorney General and CPPA have affirmed GPC as a valid opt-out signal that covered businesses must honor. The **`Sec-` prefix** marks it a [forbidden header](../02-Core-Concepts/Forbidden-and-Restricted-Headers.md) JS can't set (integrity), and it's a much narrower, legally-anchored signal than the broad, vague `DNT`.

## How does it work?

The browser (when the user enables the setting, or by default in some privacy-focused browsers/extensions) sends the header on every request. The server reads it and, for `Sec-GPC`, must apply the user's opt-out under applicable law.

```mermaid
flowchart TD
    A[User enables privacy signal in browser/extension] --> B[Browser sends header on every request]
    B --> C{Which header?}
    C -- DNT: 1 --> D[Legally non-binding<br/>→ most sites ignore (header largely removed)]
    C -- Sec-GPC: 1 --> E{Is the business subject to CCPA/CPRA etc.?}
    E -- Yes --> F[MUST honor: disable sale/sharing of personal data]
    E -- No --> G[Should treat as a privacy preference (good practice)]
```

- **Browser behavior:** Sends `Sec-GPC: 1` when the user enables Global Privacy Control (built into some browsers, or via extensions like Privacy Badger / DuckDuckGo); exposes `navigator.globalPrivacyControl` to JS. `DNT` is largely removed/deprecated; `navigator.doNotTrack` may return `null`/`"1"`/`"0"`. Both are [forbidden headers](../02-Core-Concepts/Forbidden-and-Restricted-Headers.md) (JS can't set them).
- **Server behavior:** For `Sec-GPC: 1`, a covered business must **act on it** (disable data sale/sharing, honor opt-out, reflect it in consent state). Reading `DNT` is optional and mostly historical. Servers often also expose a machine-readable policy at `/.well-known/gpc.json`.
- **Proxy/CDN behavior:** Pass the headers through; some privacy-focused CDNs/edges can inject or act on GPC.
- **Reverse proxy behavior:** Can read GPC to set downstream flags, but the *legal* action (not selling data) happens in application/ad-tech logic.

## HTTP Request Example

A user with Global Privacy Control enabled:

```http
GET /products HTTP/1.1
Host: shop.example.com
Sec-GPC: 1
DNT: 1
```

Most modern browsers send **only `Sec-GPC: 1`** (DNT removed):

```http
GET /products HTTP/1.1
Host: shop.example.com
Sec-GPC: 1
```

## HTTP Response Example

There's no dedicated response header for compliance, but a common pattern is a **well-known GPC policy document** advertising that you honor it:

```http
GET /.well-known/gpc.json HTTP/1.1
Host: shop.example.com
```

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"gpc": true, "lastUpdate": "2026-07-07"}
```

(`{"gpc": true}` declares the site honors the Global Privacy Control signal.)

## Express.js Example

```js
const express = require('express');
const app = express();

// 1) Read the GPC signal on every request and set an opt-out flag.
app.use((req, res, next) => {
  // Sec-GPC: "1" = user opts out of sale/sharing of personal info.
  req.gpcOptOut = req.headers['sec-gpc'] === '1';
  // DNT is legacy; read it only for historical/analytics purposes.
  req.dnt = req.headers['dnt'] === '1';
  next();
});

// 2) HONOR the signal: disable data sale/sharing + non-essential tracking.
app.use((req, res, next) => {
  if (req.gpcOptOut) {
    // Turn off ad-tech data sharing, third-party pixels, "sale" of data, etc.
    res.locals.analytics = 'essential-only';
    res.locals.adPersonalization = false;
    // Reflect the opt-out in your consent/CMP state and downstream systems.
    consentStore.recordOptOut(req);            // your compliance logic
  }
  next();
});

// 3) Render with tracking gated on the opt-out.
app.get('/products', (req, res) => {
  res.type('html').send(renderProducts({
    // Only load personalized ads / third-party trackers if NOT opted out.
    personalizedAds: !req.gpcOptOut,
    analytics: res.locals.analytics,
  }));
});

// 4) Advertise compliance via the well-known document.
app.get('/.well-known/gpc.json', (req, res) => {
  res.json({ gpc: true, lastUpdate: '2026-07-07' });
});

app.listen(3000);
```

Why each piece matters: for a business subject to CCPA/CPRA, step 2 is not optional politeness — **honoring `Sec-GPC: 1` is a legal obligation**, so the flag must actually flow into your ad-tech/data-sharing/consent systems (turning off third-party data sale, personalized ads, non-essential trackers). Reading the header (step 1) is trivial; the real work is *wiring the opt-out through your entire data pipeline* (CMP, tag manager, server-side integrations). The well-known document (step 4) is how you *declare* compliance. Note `DNT` is read only for legacy/analytics — you're not obligated to act on it, and it's mostly gone.

## Node.js Example

Raw `http`:

```js
const http = require('http');

http.createServer((req, res) => {
  const gpc = req.headers['sec-gpc'] === '1';   // legally significant opt-out
  const dnt = req.headers['dnt'] === '1';       // legacy, advisory

  if (req.url === '/.well-known/gpc.json') {
    res.setHeader('Content-Type', 'application/json');
    return res.end(JSON.stringify({ gpc: true, lastUpdate: '2026-07-07' }));
  }

  // Honor GPC: suppress data sale/sharing + non-essential tracking.
  const trackingAllowed = !gpc;
  res.setHeader('Content-Type', 'text/html; charset=utf-8');
  res.end(`<!doctype html>${trackingAllowed ? '<script src="/ads.js"></script>' : '<!-- tracking suppressed per GPC -->'}`);
}).listen(3000);
```

The essential move: read `Sec-GPC`, and if `1`, **suppress data sale/sharing** everywhere it would otherwise happen.

## React Example

React apps must respect GPC on the client too — where much tracking (analytics, ad pixels, third-party SDKs) actually runs:

```jsx
function useGpcOptOut() {
  // navigator.globalPrivacyControl reflects the Sec-GPC signal client-side.
  return typeof navigator !== 'undefined' && navigator.globalPrivacyControl === true;
}

function App() {
  const optedOut = useGpcOptOut();

  React.useEffect(() => {
    if (optedOut) {
      // Do NOT initialize personalized ads / data-selling third-party trackers.
      // Only load essential/first-party analytics (or none).
      disableAdPersonalization();
      // e.g. gtag('consent', 'update', { ad_storage: 'denied', ad_user_data: 'denied' });
    } else {
      initAnalytics();
    }
  }, [optedOut]);

  return <Shop personalizedAds={!optedOut} />;
}
```

Key points for React devs:
1. **Read `navigator.globalPrivacyControl`** and gate client-side tracking/ad SDKs on it — much data "sale/sharing" happens via client-side scripts, so server-side honoring alone is insufficient.
2. **Wire it into your consent tooling** (Google Consent Mode, your CMP): set consent to denied for ad storage/data when GPC is on.
3. **`navigator.doNotTrack`** exists but is legacy/unreliable; prioritize GPC.
4. **You can't set these headers** (forbidden); you only *read* the browser's signal.
5. **Treat compliance as end-to-end:** client SDKs, server integrations, and third-party tags all must respect the opt-out.

## Browser Lifecycle

1. The user enables the privacy signal (browser setting or extension); some privacy browsers send it by default.
2. The browser attaches `Sec-GPC: 1` (and, legacy, possibly `DNT: 1`) to **every** request automatically.
3. JS can read `navigator.globalPrivacyControl` / `navigator.doNotTrack` but **cannot set** the headers (forbidden).
4. The server (and client scripts) read the signal and, for GPC on covered businesses, **must apply the opt-out** (disable sale/sharing, personalized ads, non-essential tracking).
5. `DNT` is largely removed from modern browsers; don't rely on its presence.
6. There's no browser-enforced consequence — enforcement is **legal**, via privacy regulators.

## Production Use Cases

- **CCPA/CPRA compliance:** honoring `Sec-GPC: 1` as a legally-binding "do not sell/share" opt-out for covered businesses.
- **Consent management:** feeding GPC into your CMP/Consent Mode to set default consent to denied.
- **Ad-tech gating:** disabling personalized ads and third-party data sharing when GPC is on.
- **Privacy-respecting analytics:** switching to essential/first-party-only analytics under GPC.
- **Compliance advertisement:** serving `/.well-known/gpc.json` to declare you honor GPC.
- **Legacy awareness:** recognizing `DNT` when auditing old code (but not relying on it).

## Common Mistakes

- **Treating `Sec-GPC` as ignorable like `DNT`.** For covered businesses, ignoring GPC can be a **legal violation**. It's not optional.
- **Honoring it only server-side.** Much tracking runs in client JS; you must gate client-side SDKs/pixels on `navigator.globalPrivacyControl` too.
- **Confusing the two.** `DNT` = dead/voluntary; `Sec-GPC` = alive/legally-significant. Don't conflate their weight.
- **Relying on `DNT` presence.** It's largely removed; `navigator.doNotTrack` is unreliable.
- **Not wiring it through third-party tags.** Tag managers, ad SDKs, and server-side integrations must all reflect the opt-out.
- **Trying to set the headers from JS.** Forbidden/ignored.
- **Assuming it means "no analytics at all."** GPC targets *sale/sharing of personal data*; essential first-party analytics may still be permissible (check your legal guidance).
- **Fingerprinting concern with `DNT`.** Sending `DNT` added entropy for little benefit — another reason it died.

## Security Considerations

- **Legal risk is the "security" here.** The primary risk of mishandling `Sec-GPC` is **regulatory/legal** (fines, enforcement) for covered businesses, not a technical exploit. Treat compliance as a first-class requirement.
- **Privacy integrity:** the `Sec-` prefix ensures page JS can't forge/suppress the signal, so a site can't trivially strip a user's opt-out via client script.
- **Don't use it for tracking/fingerprinting.** Ironically, differentiating users by whether they send GPC could itself be a privacy harm; use it only to *honor* the preference.
- **Auditability:** record that you honored the opt-out (consent logs) for compliance evidence.
- **Not authentication/authorization:** it's a preference signal, not identity.

## Performance Considerations

- **Negligible wire cost;** tiny headers.
- **Potential *positive* performance impact:** honoring GPC often means *not* loading heavy third-party ad/tracking scripts, which can **speed up** the page for opted-out users.
- **No caching interaction of note,** though if responses differ by GPC (e.g. with/without trackers) and you cache HTML, you may need to account for it (usually tracking is client-side, so this is rare).
- **`DNT` added fingerprinting entropy** historically — a privacy cost, not a perf one.

## Reverse Proxy Considerations

Nginx can surface GPC to the app and serve the well-known doc:

```nginx
server {
  # Advertise GPC compliance.
  location = /.well-known/gpc.json {
    default_type application/json;
    return 200 '{"gpc": true, "lastUpdate": "2026-07-07"}';
  }

  location / {
    proxy_pass http://app_upstream;
    # Pass the signal through (it's a normal request header; forwarded by default).
    # Optionally set a normalized flag for the app:
    proxy_set_header X-GPC $http_sec_gpc;
  }
}
```

Key points: the proxy can *surface* the signal, but the **legal action** (not selling/sharing data) happens in application and ad-tech logic — the proxy can't "comply" on its own. Serving `/.well-known/gpc.json` is a simple, standard declaration.

## CDN Considerations

- **Pass `Sec-GPC` through** to origin; some privacy-focused edges can act on it.
- **Edge personalization/ads:** if the CDN injects ad/tracking scripts, ensure it respects GPC.
- **Well-known doc at the edge:** you can serve `/.well-known/gpc.json` from the CDN.
- **Consistency:** the opt-out must be honored across all delivery layers (edge, origin, client), not just one.

## Cloud Deployment Considerations

- **Consent platforms / CMPs:** integrate GPC into your consent management (OneTrust, Google Consent Mode, etc.) so the opt-out propagates to all tags/SDKs.
- **Ad/analytics integrations:** server-side and client-side ad-tech (GA4, Meta, etc.) must be configured to honor GPC/opt-out state.
- **Data pipelines:** downstream data sharing/sale must be suppressed for opted-out users end-to-end.
- **Legal/compliance sign-off:** GPC handling should be reviewed by privacy/legal for your applicable jurisdictions.
- **Well-known doc:** host `/.well-known/gpc.json` on your primary domains.

## Debugging

- **Chrome DevTools → Network → Headers:** check for `Sec-GPC: 1` (and legacy `DNT: 1`) on requests when the browser/extension has GPC enabled.
- **Console:** `navigator.globalPrivacyControl` (true when GPC on) and `navigator.doNotTrack` (legacy).
- **curl:** `curl -H 'Sec-GPC: 1' https://host/products` to test server handling.
- **Verify honoring:** confirm that with `Sec-GPC: 1`, personalized ads/third-party trackers/data-sharing are actually disabled (inspect what scripts load and what network calls fire).
- **Well-known doc:** `curl https://host/.well-known/gpc.json` → `{"gpc": true}`.
- **Extensions:** test with a GPC-enabling extension (Privacy Badger, DuckDuckGo) to generate the signal.

## Best Practices

- [ ] **Honor `Sec-GPC: 1`** as a legally-binding opt-out if you're subject to CCPA/CPRA (or treat it as one broadly, as good practice).
- [ ] Apply the opt-out **end-to-end**: client SDKs/pixels, server integrations, and downstream data sharing.
- [ ] Read `Sec-GPC` server-side **and** `navigator.globalPrivacyControl` client-side.
- [ ] Wire GPC into your **CMP/Consent Mode** (set ad/data consent to denied).
- [ ] Serve `/.well-known/gpc.json` (`{"gpc": true}`) to declare compliance.
- [ ] Treat **`DNT` as legacy** — read only for history/analytics; don't rely on it.
- [ ] **Log/record** opt-out honoring for compliance evidence.
- [ ] Don't try to set these headers from JS (forbidden); only read the signal.
- [ ] Get privacy/legal review for your jurisdictions.

## Related Headers

- [User-Agent](./User-Agent.md) / [Sec-CH-UA](./Sec-CH-UA.md) — other client-identity signals; UA-CH is the privacy-tiered device-info replacement (different privacy dimension).
- [Referrer-Policy](../05-Security-Headers/Referrer-Policy.md) — controls referrer leakage, another privacy control.
- [Cookie](../08-Cookies/Cookie.md) / [Set-Cookie](../08-Cookies/Set-Cookie.md) — tracking cookies are a primary thing GPC opt-out affects (`SameSite`, consent).
- [Forbidden and Restricted Headers](../02-Core-Concepts/Forbidden-and-Restricted-Headers.md) — why JS can't set `Sec-GPC`/`DNT`.
- [Custom and X- Headers](../02-Core-Concepts/Custom-and-X-Headers.md) — context on the `Sec-` prefix.

## Decision Tree

```mermaid
flowchart TD
    A[Request arrives] --> B{Sec-GPC: 1?}
    B -- Yes --> C{Business subject to CCPA/CPRA etc.?}
    C -- Yes --> D[MUST honor: disable sale/sharing,<br/>personalized ads, non-essential trackers<br/>(client + server + downstream)]
    C -- No --> E[Should honor as a privacy preference]
    B -- No --> F[Normal processing per consent state]
    A --> G{DNT: 1?}
    G -- Yes --> H[Legacy: optional, no legal obligation]
```

## Mental Model

Think of `DNT` and `Sec-GPC` as **two nearly-identical "No Soliciting" stickers on a front door — but one is just a polite note and the other is backed by a local ordinance.** `DNT` was the polite note: homeowners put it up, but salespeople had no legal reason to respect it, so almost all of them knocked anyway — and the note became so pointless (and oddly, so *identifying*) that people stopped bothering with it. `Sec-GPC` is the sticker that a *city law* recognizes: for businesses operating under that law, ignoring it isn't rude, it's a **violation** — so when they see it, they must genuinely stop selling your information to the door-to-door data brokers, stop the targeted pitches, and log that they complied. The crucial practical lesson: you must honor the legally-backed sticker **everywhere the soliciting could happen** — not just the front door (your server), but also the side gate and the mailbox (client-side ad scripts and third-party integrations), because "we stopped at the front but kept selling via the back" isn't compliance. And you can't fake the sticker for someone (the browser controls it) — you can only *read* it and act.
