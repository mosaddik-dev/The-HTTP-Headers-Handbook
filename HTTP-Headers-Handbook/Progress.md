# Progress Tracker

**Overall completion: ~65%**
**Last updated: 2026-07-07 (reconciled with actual files on disk)**

At the start of every session: read this file first, then continue from the first unchecked item. Never redo completed work unless requested.

> **Note:** This tracker was corrected on 2026-07-07. A prior session wrote ~74 substantial pages but hit a session limit before updating this file, which still read "~9%". The checklist below now reflects the files actually present on disk.

Legend: `[x]` complete · `[~]` in progress · `[ ]` not started · `[f]` covered (folded into a combined chapter, no standalone file)

---

## Part 01 — Introduction

- [x] What are HTTP Headers (2026-07-07)
- [x] Anatomy of an HTTP Message (2026-07-07)
- [x] Request vs Response Headers (2026-07-07)
- [x] End-to-End vs Hop-by-Hop Headers (2026-07-07)
- [x] How Browsers Process Headers (2026-07-07)
- [x] How Servers Process Headers (2026-07-07)
- [x] HTTP Versions and Headers (1.1 / 2 / 3) (2026-07-07)

## Part 02 — Core Concepts

- [x] Header Categories (2026-07-07)
- [x] Header Syntax and Grammar (ABNF, tokens, quoted strings) (2026-07-07)
- [x] Structured Field Values (RFC 8941) (2026-07-07)
- [x] Forbidden and Restricted Headers (2026-07-07)
- [x] Custom and X- Headers (2026-07-07)
- [x] Case Sensitivity and Ordering (2026-07-07)
- [x] Header Size Limits (2026-07-07)

## Part 03 — Request Headers

- [x] Host (2026-07-07)
- [x] Origin (2026-07-07)
- [x] Referer (2026-07-07)
- [x] User-Agent (2026-07-07)
- [x] Accept (2026-07-07)
- [x] Accept-Language (2026-07-07)
- [x] Accept-Encoding (2026-07-07) — lives in Part 10
- [x] Accept-Charset (2026-07-07)
- [x] Authorization (2026-07-07) — lives in Part 09
- [x] Cookie (2026-07-07) — lives in Part 08
- [x] Content-Type (request) (2026-07-07)
- [x] Content-Length (request) (2026-07-07) — lives in Part 04
- [x] Connection (2026-07-07)
- [x] Expect (2026-07-07)
- [x] TE (2026-07-07)
- [x] Upgrade (2026-07-07)
- [x] Sec-Fetch-Site / Mode / User / Dest (2026-07-07)
- [x] Sec-CH-UA (Client Hints) (2026-07-07)
- [x] DNT / Sec-GPC (2026-07-07)
- [x] Max-Forwards (2026-07-07)

## Part 04 — Response Headers

- [x] Content-Type (response) (2026-07-07)
- [x] Content-Length (response) (2026-07-07)
- [x] Content-Disposition (2026-07-07)
- [x] Content-Language (2026-07-07)
- [x] Content-Location (2026-07-07)
- [x] Location (2026-07-07)
- [x] Server (2026-07-07)
- [x] Date (2026-07-07)
- [x] Retry-After (2026-07-07)
- [x] Allow (2026-07-07)
- [x] Accept-Ranges (2026-07-07)
- [x] Connection (response) (2026-07-07) — covered in Part 03 Connection
- [x] Keep-Alive (2026-07-07)

## Part 05 — Security Headers

- [x] Strict-Transport-Security (2026-07-07)
- [x] Content-Security-Policy (2026-07-07)
- [x] Content-Security-Policy-Report-Only (2026-07-07)
- [x] X-Content-Type-Options (2026-07-07)
- [x] X-Frame-Options (2026-07-07)
- [x] X-XSS-Protection (2026-07-07)
- [x] Referrer-Policy (2026-07-07)
- [x] Permissions-Policy (2026-07-07)
- [x] Cross-Origin-Opener-Policy (2026-07-07)
- [x] Cross-Origin-Embedder-Policy (2026-07-07)
- [x] Cross-Origin-Resource-Policy (2026-07-07)
- [x] X-DNS-Prefetch-Control (2026-07-07)
- [x] X-Permitted-Cross-Domain-Policies (2026-07-07)
- [x] Reporting-Endpoints / Report-To (2026-07-07)
- [x] NEL (Network Error Logging) (2026-07-07)

## Part 06 — Caching Headers

- [x] Cache-Control (2026-07-07)
- [x] Expires (2026-07-07)
- [x] ETag (2026-07-07)
- [x] Last-Modified (2026-07-07)
- [x] Age (2026-07-07)
- [x] Vary (2026-07-07)
- [x] Pragma (2026-07-07)
- [x] Surrogate-Control / Surrogate-Key (CDN) (2026-07-07)

## Part 07 — CORS

- [x] CORS Overview (2026-07-07)
- [x] Origin (2026-07-07) — lives in Part 03
- [x] Access-Control-Allow-Origin (2026-07-07)
- [x] Access-Control-Allow-Methods (2026-07-07)
- [x] Access-Control-Allow-Headers (2026-07-07)
- [x] Access-Control-Allow-Credentials (2026-07-07)
- [x] Access-Control-Expose-Headers (2026-07-07)
- [x] Access-Control-Max-Age (2026-07-07)
- [x] Access-Control-Request-Method (2026-07-07)
- [x] Access-Control-Request-Headers (2026-07-07)

## Part 08 — Cookies

- [x] Cookies Overview (2026-07-07)
- [x] Set-Cookie (2026-07-07)
- [x] Cookie (2026-07-07)
- [f] Cookie Attributes (SameSite, Secure, HttpOnly, Domain, Path, Max-Age) — folded into Set-Cookie
- [f] Sessions vs Stateless Tokens — folded into Cookies-Overview / Authentication-Overview

## Part 09 — Authentication

- [x] Authentication Overview (2026-07-07)
- [x] Authorization (2026-07-07)
- [x] WWW-Authenticate (2026-07-07)
- [x] Proxy-Authorization (2026-07-07)
- [x] Proxy-Authenticate (2026-07-07)
- [f] Bearer / JWT / OAuth flows — folded into Authentication-Overview / Authorization

## Part 10 — Compression

- [x] Accept-Encoding (2026-07-07)
- [x] Content-Encoding (2026-07-07)
- [x] Transfer-Encoding (2026-07-07)
- [x] Content-Length vs Transfer-Encoding (2026-07-07)

## Part 11 — Content Negotiation

- [x] Content Negotiation Overview (2026-07-07)
- [x] Accept (2026-07-07) — cross-ref, lives in Part 03
- [x] Accept-Language (2026-07-07) — lives in Part 03
- [x] Accept-Encoding (cross-ref) (2026-07-07) — lives in Part 10
- [x] Vary (cross-ref) (2026-07-07) — lives in Part 06

## Part 12 — Conditional Requests

- [x] Conditional Requests Overview (2026-07-07)
- [x] If-Match (2026-07-07)
- [x] If-None-Match (2026-07-07)
- [x] If-Modified-Since (2026-07-07)
- [x] If-Unmodified-Since (2026-07-07)
- [x] If-Range (2026-07-07)

## Part 13 — Range Requests

- [x] Range Requests Overview (2026-07-07)
- [x] Range (2026-07-07)
- [x] Content-Range (2026-07-07)
- [ ] Accept-Ranges (cross-ref)

## Part 14 — Proxies

- [x] Proxies Overview (2026-07-07)
- [x] Via (2026-07-07)
- [x] Forwarded (2026-07-07)
- [x] X-Forwarded-For (2026-07-07)
- [x] X-Forwarded-Proto (2026-07-07)
- [x] X-Forwarded-Host (2026-07-07)
- [x] X-Real-IP (2026-07-07)

## Part 15 — CDNs

- [x] CDN Caching Overview (2026-07-07)
- [x] Cache Keys and Vary (2026-07-07)
- [x] Cloudflare-specific headers (2026-07-07)
- [x] CDN debugging headers (2026-07-07)

## Part 16 — Reverse Proxies

- [x] Reverse Proxy Overview (2026-07-07)
- [f] Nginx header handling — folded into Reverse-Proxy-Overview
- [f] Header rewriting and pitfalls — folded into Reverse-Proxy-Overview

## Part 17 — Express.js

- [x] How Express reads request headers (2026-07-07) — Headers-in-Express.md
- [x] How Express writes response headers (2026-07-07) — Headers-in-Express.md
- [x] trust proxy and X-Forwarded-* (2026-07-07) — Headers-in-Express.md
- [x] helmet and security headers (2026-07-07) — Headers-in-Express.md

## Part 18 — React

- [x] How React apps depend on headers (2026-07-07) — Headers-and-React.md
- [x] SSR / Next.js header handling (2026-07-07) — Headers-and-React.md
- [x] fetch/axios and headers (2026-07-07) — Headers-and-React.md

## Part 19 — Debugging

- [x] Chrome DevTools (2026-07-07) — Debugging-Headers.md
- [x] curl (2026-07-07) — Debugging-Headers.md
- [x] Postman / Bruno (2026-07-07) — Debugging-Headers.md
- [x] Node.js and Express logging (2026-07-07) — Debugging-Headers.md

## Part 20 — Real-World Architectures

- [x] Browser → CDN → Reverse Proxy → App → DB header flow (2026-07-07) — End-to-End-Header-Flow.md
- [x] Auth architecture end-to-end (2026-07-07)
- [x] Caching architecture end-to-end (2026-07-07)

## Part 21 — Best Practices

- [x] Security hardening checklist (2026-07-07) — Security-Hardening.md
- [ ] Caching strategy checklist
- [ ] Anti-patterns

## Part 22 — Reference

- [x] Cheat Sheet (by category) (2026-07-07)
- [x] Status Code ↔ Header map (2026-07-07)
- [ ] Glossary
- [ ] Master Decision Tree

---

## Remaining

Genuine gaps (standalone pages not yet written), in the priority order suggested earlier:

**High-value headers still missing**
- Part 06: Vary, Age (Vary is heavily cross-referenced by CDN/negotiation chapters — worth prioritizing)
- Part 07: Access-Control-Allow-Credentials, Access-Control-Request-Method, Access-Control-Request-Headers (completes the CORS set)
- Part 10: Transfer-Encoding (+ Content-Length vs Transfer-Encoding)
- Part 12: If-None-Match, If-Modified-Since, If-Unmodified-Since, If-Range (only If-Match written so far)
- Part 13: Content-Range (Range exists; its counterpart is missing)
- Part 14: Forwarded, X-Forwarded-For, X-Forwarded-Proto, X-Forwarded-Host, X-Real-IP (only Overview + Via exist — a big gap given how much other chapters reference these)

**Response/request completeness**
- Part 02: Header Size Limits
- Part 03: Accept-Language, Accept-Charset, TE, Sec-CH-UA, DNT/Sec-GPC, Max-Forwards
- Part 04: Content-Language, Content-Location, Retry-After, Allow, Accept-Ranges, Keep-Alive
- Part 05: COEP, CORP, X-DNS-Prefetch-Control, X-Permitted-Cross-Domain-Policies, Reporting-Endpoints/Report-To, NEL
- Part 06: Pragma, Surrogate-Control/Surrogate-Key
- Part 09: Proxy-Authenticate
- Part 11: Accept-Language page, Vary cross-ref

**Chapters / reference**
- Part 15: Cloudflare-specific headers, CDN debugging headers
- Part 20: Auth architecture end-to-end, Caching architecture end-to-end
- Part 21: Caching strategy checklist, Anti-patterns
- Part 22: Glossary (referenced in SUMMARY.md but file is missing), Master Decision Tree

**Housekeeping**
- SUMMARY.md is missing several already-written pages (e.g. X-XSS-Protection, CSP-Report-Only, Proxy-Authorization, If-Match, Cache-Keys-and-Vary, Status-Code-Header-Map). It should be synced with the files on disk.
