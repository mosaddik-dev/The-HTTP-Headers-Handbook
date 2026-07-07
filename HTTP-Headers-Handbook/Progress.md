# Progress Tracker

**Overall completion: ~9%**
**Last updated: 2026-07-07**

At the start of every session: read this file first, then continue from the first unchecked item. Never redo completed work unless requested.

Legend: `[x]` complete · `[~]` in progress · `[ ]` not started

---

## Part 01 — Introduction

- [x] What are HTTP Headers (2026-07-07)
- [x] Anatomy of an HTTP Message (2026-07-07)
- [x] Request vs Response Headers (2026-07-07)
- [x] End-to-End vs Hop-by-Hop Headers (2026-07-07)
- [ ] How Browsers Process Headers
- [ ] How Servers Process Headers
- [ ] HTTP Versions and Headers (1.1 / 2 / 3)

## Part 02 — Core Concepts

- [ ] Header Categories
- [ ] Header Syntax and Grammar (ABNF, tokens, quoted strings)
- [ ] Structured Field Values (RFC 8941)
- [ ] Forbidden and Restricted Headers
- [ ] Custom and X- Headers
- [ ] Case Sensitivity and Ordering
- [ ] Header Size Limits

## Part 03 — Request Headers

- [ ] Host
- [ ] Origin
- [ ] Referer
- [ ] User-Agent
- [ ] Accept
- [ ] Accept-Language
- [ ] Accept-Encoding
- [ ] Accept-Charset
- [ ] Authorization
- [ ] Cookie
- [ ] Content-Type (request)
- [ ] Content-Length (request)
- [ ] Connection
- [ ] Expect
- [ ] TE
- [ ] Upgrade
- [ ] Sec-Fetch-Site / Mode / User / Dest
- [ ] Sec-CH-UA (Client Hints)
- [ ] DNT / Sec-GPC
- [ ] Max-Forwards

## Part 04 — Response Headers

- [ ] Content-Type (response)
- [ ] Content-Length (response)
- [ ] Content-Disposition
- [ ] Content-Language
- [ ] Content-Location
- [ ] Location
- [ ] Server
- [ ] Date
- [ ] Retry-After
- [ ] Allow
- [ ] Accept-Ranges
- [ ] Connection (response)
- [ ] Keep-Alive

## Part 05 — Security Headers

- [ ] Strict-Transport-Security
- [ ] Content-Security-Policy
- [ ] Content-Security-Policy-Report-Only
- [ ] X-Content-Type-Options
- [ ] X-Frame-Options
- [ ] X-XSS-Protection
- [ ] Referrer-Policy
- [ ] Permissions-Policy
- [ ] Cross-Origin-Opener-Policy
- [ ] Cross-Origin-Embedder-Policy
- [ ] Cross-Origin-Resource-Policy
- [ ] X-DNS-Prefetch-Control
- [ ] X-Permitted-Cross-Domain-Policies
- [ ] Reporting-Endpoints / Report-To
- [ ] NEL (Network Error Logging)

## Part 06 — Caching Headers

- [ ] Cache-Control
- [ ] Expires
- [ ] ETag
- [ ] Last-Modified
- [ ] Age
- [ ] Vary
- [ ] Pragma
- [ ] Surrogate-Control / Surrogate-Key (CDN)

## Part 07 — CORS

- [ ] CORS Overview
- [ ] Origin
- [ ] Access-Control-Allow-Origin
- [ ] Access-Control-Allow-Methods
- [ ] Access-Control-Allow-Headers
- [ ] Access-Control-Allow-Credentials
- [ ] Access-Control-Expose-Headers
- [ ] Access-Control-Max-Age
- [ ] Access-Control-Request-Method
- [ ] Access-Control-Request-Headers

## Part 08 — Cookies

- [ ] Cookies Overview
- [ ] Set-Cookie
- [ ] Cookie
- [ ] Cookie Attributes (SameSite, Secure, HttpOnly, Domain, Path, Max-Age)
- [ ] Sessions vs Stateless Tokens

## Part 09 — Authentication

- [ ] Authentication Overview
- [ ] Authorization
- [ ] WWW-Authenticate
- [ ] Proxy-Authorization
- [ ] Proxy-Authenticate
- [ ] Bearer / JWT / OAuth flows

## Part 10 — Compression

- [ ] Accept-Encoding
- [ ] Content-Encoding
- [ ] Transfer-Encoding
- [ ] Content-Length vs Transfer-Encoding

## Part 11 — Content Negotiation

- [ ] Content Negotiation Overview
- [ ] Accept
- [ ] Accept-Language
- [ ] Accept-Encoding (cross-ref)
- [ ] Vary (cross-ref)

## Part 12 — Conditional Requests

- [ ] Conditional Requests Overview
- [ ] If-Match
- [ ] If-None-Match
- [ ] If-Modified-Since
- [ ] If-Unmodified-Since
- [ ] If-Range

## Part 13 — Range Requests

- [ ] Range Requests Overview
- [ ] Range
- [ ] Content-Range
- [ ] Accept-Ranges (cross-ref)

## Part 14 — Proxies

- [ ] Proxies Overview
- [ ] Via
- [ ] Forwarded
- [ ] X-Forwarded-For
- [ ] X-Forwarded-Proto
- [ ] X-Forwarded-Host
- [ ] X-Real-IP

## Part 15 — CDNs

- [ ] CDN Caching Overview
- [ ] Cache Keys and Vary
- [ ] Cloudflare-specific headers
- [ ] CDN debugging headers

## Part 16 — Reverse Proxies

- [ ] Reverse Proxy Overview
- [ ] Nginx header handling
- [ ] Header rewriting and pitfalls

## Part 17 — Express.js

- [ ] How Express reads request headers
- [ ] How Express writes response headers
- [ ] trust proxy and X-Forwarded-*
- [ ] helmet and security headers

## Part 18 — React

- [ ] How React apps depend on headers
- [ ] SSR / Next.js header handling
- [ ] fetch/axios and headers

## Part 19 — Debugging

- [ ] Chrome DevTools
- [ ] curl
- [ ] Postman / Bruno
- [ ] Node.js and Express logging

## Part 20 — Real-World Architectures

- [ ] Browser → CDN → Reverse Proxy → App → DB header flow
- [ ] Auth architecture end-to-end
- [ ] Caching architecture end-to-end

## Part 21 — Best Practices

- [ ] Security hardening checklist
- [ ] Caching strategy checklist
- [ ] Anti-patterns

## Part 22 — Reference

- [ ] Cheat Sheet (by category)
- [ ] Status Code ↔ Header map
- [ ] Glossary
- [ ] Master Decision Tree

---

## Remaining

Everything unchecked above. Priority order for next sessions:
1. Finish Part 01 (browsers/servers/versions)
2. Part 02 Core Concepts
3. High-value headers: Content-Type, Cache-Control, Set-Cookie, Authorization, CSP, Access-Control-Allow-Origin
4. Remaining request/response headers
5. Category overview chapters (14–21)
6. Reference (22)
