# Reporting-Endpoints / Report-To

## Quick Summary

`Reporting-Endpoints` (and its older predecessor `Report-To`) are **response** headers that define **named destinations to which the browser sends out-of-band reports** about things that happen on your pages — [CSP](./Content-Security-Policy.md) violations, [COEP](./Cross-Origin-Embedder-Policy.md)/[COOP](./Cross-Origin-Opener-Policy.md) breakages, deprecation warnings, browser interventions, crashes, and network errors ([NEL](./NEL.md)). They implement the **Reporting API**: instead of each policy header carrying its own report URL, you declare named endpoints once — `Reporting-Endpoints: csp-endpoint="https://reports.example.com/csp", coep="https://reports.example.com/coep"` — and other headers *refer to them by name* (e.g. CSP's `report-to csp-endpoint`). The browser then POSTs batched JSON reports to those URLs asynchronously, out of band, so you get **production telemetry on policy violations and browser-observed problems** without instrumenting client code. `Report-To` is the legacy JSON-based version (still needed for [NEL](./NEL.md) and older browsers); `Reporting-Endpoints` (the "Reporting API v1") is the modern, simpler replacement for document-scoped reports. These headers don't *enforce* anything — they're the **observability backbone** that makes security headers safely deployable: you roll out policies in report-only mode, watch the reports, then enforce.

## What problem does this header solve?

Security and correctness headers ([CSP](./Content-Security-Policy.md), [COEP](./Cross-Origin-Embedder-Policy.md), Permissions-Policy, etc.) are powerful but *dangerous to deploy blind*: a too-strict CSP can break your site for real users, and you won't know unless something surfaces the breakage. Historically, CSP had its own `report-uri` directive, but every reporting-capable feature reinventing its own URL was fragmented and inconsistent. And beyond policy violations, browsers observe things your server never sees — deprecated API usage, browser interventions (e.g. blocking a slow script), crashes, and **network errors that never reached your server at all** ([NEL](./NEL.md), e.g. DNS failures, TLS errors, connection resets on the way *to* you).

The Reporting API + `Reporting-Endpoints`/`Report-To` solve this by providing **one unified, named mechanism** for the browser to deliver all these reports out-of-band. You declare endpoints once and every feature routes to them by name. This gives you:
- **Safe policy rollout:** deploy [CSP](./Content-Security-Policy.md)/[COEP](./Cross-Origin-Embedder-Policy.md) in *report-only* mode and collect violations before enforcing.
- **Production visibility** into breakages, deprecations, and interventions you'd otherwise never learn about.
- **Client-invisible network diagnostics** via [NEL](./NEL.md) — errors that happen *before* a request reaches your logs.

## Why was it introduced?

The **Reporting API** (W3C) was created to unify the browser's various ad-hoc reporting mechanisms. CSP originally used `report-uri`; the first-generation Reporting API introduced **`Report-To`** (a JSON header defining named endpoint groups with load-balancing/failover), which CSP's `report-to` directive and [NEL](./NEL.md) then referenced. `Report-To` proved more complex than needed for the common case, so **Reporting API v1** introduced the simpler **`Reporting-Endpoints`** header (`name="url"` pairs) for document-scoped reports, which modern browsers prefer. `Report-To` remains relevant because [NEL](./NEL.md) still depends on it and for broader/older support. The whole family exists because *observability is a prerequisite for safely adopting strict security policies* — without a report channel, you can't tell whether a policy protects users or breaks them.

## How does it work?

You declare named endpoints in `Reporting-Endpoints` (or endpoint *groups* in `Report-To`). Other headers reference an endpoint by name. When a relevant event occurs, the browser queues a JSON report and POSTs it (batched, asynchronously, with `Content-Type: application/reports+json`) to the named URL.

```mermaid
sequenceDiagram
    participant B as Browser
    participant P as Your page
    participant R as Report collector
    P-->>B: Reporting-Endpoints: csp="https://r.example/csp"<br/>Content-Security-Policy-Report-Only: ...; report-to csp
    Note over B: User loads page; a blocked resource<br/>violates the (report-only) CSP
    B->>R: POST /csp (application/reports+json)<br/>[{type:"csp-violation", body:{...}}]
    Note over R: Aggregate → dashboards/alerts →<br/>decide when to enforce
```

- **Browser behavior:** Parses `Reporting-Endpoints`/`Report-To`, associates named endpoints, and delivers queued reports (CSP, COEP/COOP, deprecation, intervention, crash, NEL) to the matching URL as batched JSON POSTs — out of band, not blocking the page.
- **Server behavior:** The origin sets the endpoint header(s) and references them from policy headers; a *collector* endpoint receives and stores the JSON reports.
- **Report collector:** Must accept `POST` with `Content-Type: application/reports+json`, and — if cross-origin — handle CORS (the browser sends these as CORS requests to cross-origin endpoints).
- **Proxy/CDN behavior:** Pass the headers through; the collector is often a separate service or a third-party (Report URI, Sentry, etc.).
- **Reverse proxy behavior:** A central place to inject `Reporting-Endpoints` consistently.

## HTTP Request Example

These are **response** headers; there's no request-side form. The browser's *report delivery* is itself an HTTP request to your collector:

```http
POST /csp HTTP/1.1
Host: reports.example.com
Content-Type: application/reports+json

[{"age":10,"type":"csp-violation","url":"https://app.example.com/",
  "body":{"blockedURL":"https://evil.cdn/x.js","effectiveDirective":"script-src",
          "disposition":"report","documentURL":"https://app.example.com/"}}]
```

## HTTP Response Example

Modern `Reporting-Endpoints` (v1) plus a report-only CSP that references it:

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=utf-8
Reporting-Endpoints: csp-endpoint="https://reports.example.com/csp", default="https://reports.example.com/default"
Content-Security-Policy-Report-Only: default-src 'self'; script-src 'self'; report-to csp-endpoint
```

Legacy `Report-To` (JSON groups) — still used by [NEL](./NEL.md):

```http
HTTP/1.1 200 OK
Report-To: {"group":"default","max_age":10886400,"endpoints":[{"url":"https://reports.example.com/default"}]}
NEL: {"report_to":"default","max_age":10886400,"include_subdomains":true}
```

Wiring multiple features to endpoints:

```http
HTTP/1.1 200 OK
Reporting-Endpoints: main="https://reports.example.com/main"
Content-Security-Policy: default-src 'self'; report-to main
Cross-Origin-Embedder-Policy-Report-Only: require-corp; report-to="main"
```

## Express.js Example

```js
const express = require('express');
const app = express();

// 1) Declare named report endpoints once, then reference them from policy headers.
app.use((req, res, next) => {
  res.set('Reporting-Endpoints',
    'csp-endpoint="https://reports.example.com/csp", ' +
    'coep-endpoint="https://reports.example.com/coep", ' +
    'default="https://reports.example.com/default"');

  // Report-only CSP → collect violations WITHOUT breaking the page.
  res.set('Content-Security-Policy-Report-Only',
    "default-src 'self'; script-src 'self' https://cdn.example.com; report-to csp-endpoint");

  // COEP report-only to audit before enforcing cross-origin isolation.
  res.set('Cross-Origin-Embedder-Policy-Report-Only', 'require-corp; report-to="coep-endpoint"');
  next();
});

// 2) A report COLLECTOR endpoint (could also be a third-party service).
//    Browsers POST application/reports+json (an array of reports).
app.post('/reports/:type',
  express.json({ type: ['application/reports+json', 'application/json'] }),
  (req, res) => {
    const reports = Array.isArray(req.body) ? req.body : [req.body];
    for (const r of reports) {
      // Persist / forward to your logging/alerting (e.g. by r.type).
      console.log('report', req.params.type, r.type, r.body);
    }
    res.status(204).end();   // acknowledge; no body needed.
  }
);

// 3) If the collector is a DIFFERENT origin, it must handle CORS for the POST.
app.options('/reports/:type', (req, res) => {
  res.set('Access-Control-Allow-Origin', req.headers.origin || '*');
  res.set('Access-Control-Allow-Methods', 'POST, OPTIONS');
  res.set('Access-Control-Allow-Headers', 'Content-Type');
  res.status(204).end();
});

app.listen(3000);
```

Why each piece matters: declaring endpoints **once** in `Reporting-Endpoints` (route 1) and referencing them by name (`report-to csp-endpoint`) is the whole ergonomic win — you don't repeat URLs across every policy. Pairing with **report-only** variants is the safe-rollout pattern: you see exactly what a strict [CSP](./Content-Security-Policy.md)/[COEP](./Cross-Origin-Embedder-Policy.md) *would* break, via real user reports, before enforcing. The collector (route 2) must parse `application/reports+json` (an *array* of reports) — a common bug is expecting a single object. If the collector is cross-origin (route 3), the browser sends the report as a CORS request, so the collector needs CORS handling or reports silently fail to deliver.

## Node.js Example

Raw `http` collector:

```js
const http = require('http');

http.createServer((req, res) => {
  if (req.method === 'POST' && req.url.startsWith('/reports')) {
    let body = '';
    req.on('data', c => (body += c));
    req.on('end', () => {
      try {
        const reports = JSON.parse(body || '[]'); // array of report objects
        reports.forEach(r => console.log('report:', r.type, r.body?.effectiveDirective || ''));
      } catch { /* ignore malformed */ }
      res.statusCode = 204;
      res.end();
    });
    return;
  }
  // Document response wiring the endpoint:
  res.setHeader('Reporting-Endpoints', 'csp="/reports/csp"');
  res.setHeader('Content-Security-Policy-Report-Only', "default-src 'self'; report-to csp");
  res.setHeader('Content-Type', 'text/html; charset=utf-8');
  res.end('<!doctype html><script src="https://evil.cdn/x.js"></script>');
}).listen(3000);
```

The two halves: emit `Reporting-Endpoints` + a `report-to`-referencing policy on documents, and run a collector that accepts the batched JSON POSTs.

## React Example

React doesn't set these headers (server-side), but the Reporting API is how you gain **production visibility** into problems in a React app that client-side error tracking misses:

1. **CSP violations from third-party/injected scripts.** A React app pulling in analytics, chat widgets, or ads often trips CSP; report-only + `Reporting-Endpoints` shows you which resources violate before you enforce and break the app.

2. **Deprecations/interventions.** The browser reports deprecated API usage and interventions (e.g. a blocked autoplay or a slow-script kill) — signals your React error boundaries never catch.

3. **You can also read some reports client-side** via the `ReportingObserver` API (complementary to the headers):

```jsx
React.useEffect(() => {
  if ('ReportingObserver' in window) {
    const observer = new ReportingObserver((reports) => {
      reports.forEach(r => console.warn('browser report', r.type, r.body));
    }, { buffered: true });
    observer.observe();
    return () => observer.disconnect();
  }
}, []);
```

4. **The header-based path is server-config**, and it's what captures reports even when no JS runs (e.g. a CSP that blocked your main bundle) — a case client-side observers can't cover.

## Browser Lifecycle

1. The document response includes `Reporting-Endpoints` (and/or `Report-To`), defining named URLs.
2. Policy headers ([CSP](./Content-Security-Policy.md), [COEP](./Cross-Origin-Embedder-Policy.md), Permissions-Policy) reference an endpoint by name (`report-to <name>`).
3. When a reportable event occurs (violation, deprecation, intervention, crash, network error), the browser **queues** a report.
4. The browser **batches and POSTs** queued reports asynchronously (out of band) to the named endpoint as `application/reports+json`, with retries/failover (esp. `Report-To` groups) and CORS for cross-origin endpoints.
5. Delivery is best-effort and can be delayed/coalesced; it never blocks the page.
6. `ReportingObserver` (JS) can observe some report types in-page as a complement.

## Production Use Cases

- **Safe [CSP](./Content-Security-Policy.md) rollout:** report-only + endpoint → collect violations → tighten → enforce.
- **[COEP](./Cross-Origin-Embedder-Policy.md)/[COOP](./Cross-Origin-Opener-Policy.md) migration:** audit what cross-origin-isolation would break before enforcing.
- **[NEL](./NEL.md) network diagnostics:** capture DNS/TLS/connection errors that never reach your server (requires `Report-To`).
- **Deprecation/intervention monitoring:** learn when browsers deprecate APIs you use or intervene on your pages.
- **Crash reporting:** collect browser crash reports for reliability tracking.
- **Third-party integration:** point endpoints at Report URI, Sentry, or a homegrown collector for dashboards/alerts.

## Common Mistakes

- **Collector not parsing an array.** Reports arrive as a JSON *array* with `Content-Type: application/reports+json`; expecting a single object drops data.
- **Cross-origin collector without CORS.** Browsers send cross-origin reports as CORS requests; missing CORS = silent non-delivery.
- **Expecting real-time delivery.** Reports are batched/delayed/best-effort — not a synchronous log.
- **Using `Report-To` where `Reporting-Endpoints` suffices (or vice-versa).** Prefer `Reporting-Endpoints` for document reports; keep `Report-To` for [NEL](./NEL.md)/legacy.
- **Forgetting `report-to` in the policy header.** Declaring endpoints without referencing them means no reports flow. (Note the confusingly-similar CSP `report-uri` legacy directive — modern CSP uses `report-to <name>`.)
- **Report flooding.** A broken policy can generate huge report volumes; rate-limit/sample at the collector.
- **Leaking sensitive data in reports.** Report bodies can contain URLs/paths; secure and access-control your collector.

## Security Considerations

- **The collector is an ingestion endpoint — harden it.** It receives attacker-influenceable data (blocked URLs, sample scripts); validate, rate-limit, and don't blindly render/execute report contents (stored-XSS risk in dashboards).
- **Reports can contain sensitive info.** URLs, query strings, and paths in violation reports may include tokens or PII; restrict access to the collector and consider redaction.
- **DoS via report floods.** A misconfigured/attacked page can generate massive report traffic; sample and rate-limit.
- **CORS/exposure:** cross-origin collectors need CORS; be deliberate about which origins may report.
- **Not an enforcement mechanism:** these headers only *observe*; enforcement is done by the policy headers themselves. Report-only tells you what *would* happen, not what *is* blocked.
- **Privacy:** [NEL](./NEL.md) reports reveal client network conditions; disclose per your privacy policy.

## Performance Considerations

- **Out-of-band and non-blocking:** report delivery doesn't slow page load; it's batched and asynchronous.
- **Collector scale:** high-traffic sites can generate large report volumes — size/sample your collector accordingly.
- **`Report-To` grouping** supports load-balancing/failover across multiple endpoints for resilience.
- **Negligible header cost;** the operational cost is in the collector, not the wire.

## Reverse Proxy Considerations

Nginx injecting `Reporting-Endpoints` and referencing it from policies:

```nginx
server {
  location / {
    proxy_pass http://app_upstream;
    add_header Reporting-Endpoints 'csp="https://reports.example.com/csp", default="https://reports.example.com/default"' always;
    add_header Content-Security-Policy-Report-Only "default-src 'self'; report-to csp" always;
  }

  # A local collector proxied to your ingestion service:
  location /reports/ {
    proxy_pass http://report_collector;
  }
}
```

Key points: a proxy is a convenient place to attach `Reporting-Endpoints` and report-only policies site-wide. Ensure the collector path handles the `application/reports+json` POSTs (and CORS if cross-origin).

## CDN Considerations

- **Header injection at the edge:** CDNs can add `Reporting-Endpoints`/policy headers consistently across a property.
- **Third-party collectors:** many sites point endpoints at SaaS collectors (Report URI, Sentry) — ensure those are reachable and CORS-correct.
- **[NEL](./NEL.md) at the edge:** CDNs sometimes offer NEL/reporting integrations to capture edge-observed network errors.
- **Consistency:** apply the same reporting config across all responses so coverage is complete.

## Cloud Deployment Considerations

- **Managed hosts (Vercel/Netlify):** set `Reporting-Endpoints`/policies via `_headers`/config; run a serverless collector or use a SaaS.
- **Collector as a service:** a small serverless function accepting `application/reports+json` is a common pattern; ensure it scales and rate-limits.
- **Third-party services:** Report URI, Sentry, Datadog, etc. accept these reports directly — often simpler than self-hosting.
- **API Gateways/LBs:** pass headers through; route the collector path appropriately.

## Debugging

- **Chrome DevTools → Application → Reporting API / Frames:** shows configured endpoints and generated reports; the Network tab shows the outgoing report POSTs.
- **Report-only first:** deploy `Content-Security-Policy-Report-Only`/`Cross-Origin-Embedder-Policy-Report-Only` and watch reports arrive without breaking users.
- **curl (verify headers):** `curl -sD - -o /dev/null https://app/ | grep -i 'reporting-endpoints\|report-to\|report-only'`.
- **Test the collector:** `curl -X POST -H 'Content-Type: application/reports+json' -d '[{"type":"csp-violation","body":{}}]' https://reports.example.com/csp` and confirm it accepts/stores.
- **ReportingObserver:** log reports client-side to see what the browser is generating.
- **Cross-origin check:** confirm the collector returns proper CORS so cross-origin reports deliver.

## Best Practices

- [ ] Declare endpoints once with `Reporting-Endpoints` and reference them by name from policy headers (`report-to <name>`).
- [ ] Keep `Report-To` where you need [NEL](./NEL.md) or broader/older support.
- [ ] Roll out [CSP](./Content-Security-Policy.md)/[COEP](./Cross-Origin-Embedder-Policy.md) in **report-only** first, watch reports, then enforce.
- [ ] Build a collector that parses `application/reports+json` **arrays** and returns `204`.
- [ ] Handle **CORS** on cross-origin collectors, or reports won't deliver.
- [ ] **Rate-limit/sample** and secure the collector; treat report contents as untrusted (avoid dashboard XSS).
- [ ] Redact/limit sensitive data (URLs with tokens/PII) in stored reports.
- [ ] Alert on spikes (a surge often means a real breakage or attack).

## Related Headers

- [Content-Security-Policy](./Content-Security-Policy.md) / [Content-Security-Policy-Report-Only](./Content-Security-Policy.md) — the biggest report source; uses `report-to <name>`.
- [Cross-Origin-Embedder-Policy](./Cross-Origin-Embedder-Policy.md) / [Cross-Origin-Opener-Policy](./Cross-Origin-Opener-Policy.md) — report their violations here (report-only rollout).
- [NEL](./NEL.md) — Network Error Logging; depends on `Report-To` for its endpoint.
- [Permissions-Policy](./Permissions-Policy.md) — can also route violation reports.
- [Access-Control-Allow-Origin](../07-CORS/Access-Control-Allow-Origin.md) — needed on a cross-origin collector.
- [Security Hardening](../21-Best-Practices/Security-Hardening.md) — where reporting fits in a rollout strategy.

## Decision Tree

```mermaid
flowchart TD
    A[Want telemetry on policy/network issues?] --> B{Which reports?}
    B -- CSP/COEP/COOP/deprecation/intervention --> C[Reporting-Endpoints + report-to in policies]
    B -- Network errors (NEL) --> D[Report-To (required by NEL) + NEL header]
    C --> E[Deploy policies in report-only first]
    D --> E
    E --> F[Collector: parse application/reports+json array,<br/>handle CORS, rate-limit, secure]
    F --> G[Aggregate → alert → tighten → enforce]
```

## Mental Model

Think of `Reporting-Endpoints`/`Report-To` as **posting a return address for incident reports on a construction site** before you tighten the safety rules. When you introduce strict new rules ([CSP](./Content-Security-Policy.md), isolation), you don't slam them into force and hope — you first put up signs saying "report any problem to *this office*" (`Reporting-Endpoints: csp="..."`), then run a **trial period** where the rules are posted but not enforced (report-only): workers keep working, but every time something *would have* violated a rule, an incident slip is quietly mailed to your office. You collect the slips, see exactly what would break ("the north crane trips the new rule 40 times a day"), fix or whitelist it, and *only then* make the rules binding. The clever part is the unified return address: instead of every safety rule having its own mailbox, they all reference the same named offices, and the mail (`application/reports+json`) arrives batched, out-of-band, without ever interrupting the work. And a special class of slips ([NEL](./NEL.md)) reports accidents that happen *on the road to your site* — trucks that never arrived (DNS/TLS/connection failures) — problems your on-site logs could never have recorded because the truck never made it through the gate.
