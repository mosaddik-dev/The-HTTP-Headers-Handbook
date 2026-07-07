# Table of Contents

## 01 — Introduction
- [What are HTTP Headers](./01-Introduction/What-are-HTTP-Headers.md)
- [Anatomy of an HTTP Message](./01-Introduction/Anatomy-of-an-HTTP-Message.md)
- [Request vs Response Headers](./01-Introduction/Request-vs-Response-Headers.md)
- [End-to-End vs Hop-by-Hop Headers](./01-Introduction/End-to-End-vs-Hop-by-Hop-Headers.md)
- [How Browsers Process Headers](./01-Introduction/How-Browsers-Process-Headers.md)
- [How Servers Process Headers](./01-Introduction/How-Servers-Process-Headers.md)
- [HTTP Versions and Headers](./01-Introduction/HTTP-Versions-and-Headers.md)

## 02 — Core Concepts
- [Header Categories](./02-Core-Concepts/Header-Categories.md)
- [Header Syntax and Grammar](./02-Core-Concepts/Header-Syntax-and-Grammar.md)
- [Structured Field Values](./02-Core-Concepts/Structured-Field-Values.md)
- [Forbidden and Restricted Headers](./02-Core-Concepts/Forbidden-and-Restricted-Headers.md)
- [Custom and X- Headers](./02-Core-Concepts/Custom-and-X-Headers.md)
- [Case Sensitivity and Ordering](./02-Core-Concepts/Case-Sensitivity-and-Ordering.md)
- [Header Size Limits](./02-Core-Concepts/Header-Size-Limits.md)

## 03 — Request Headers
- [Host](./03-Request-Headers/Host.md)
- [Origin](./03-Request-Headers/Origin.md)
- [Referer](./03-Request-Headers/Referer.md)
- [User-Agent](./03-Request-Headers/User-Agent.md)
- [Accept](./03-Request-Headers/Accept.md)
- [Accept-Language](./03-Request-Headers/Accept-Language.md)
- [Accept-Encoding](./03-Request-Headers/Accept-Encoding.md)
- [Authorization](./03-Request-Headers/Authorization.md)
- [Cookie](./03-Request-Headers/Cookie.md)
- [Content-Type (request)](./03-Request-Headers/Content-Type.md)
- [Content-Length (request)](./03-Request-Headers/Content-Length.md)
- [Connection](./03-Request-Headers/Connection.md)
- [Expect](./03-Request-Headers/Expect.md)
- [Upgrade](./03-Request-Headers/Upgrade.md)
- [Sec-Fetch-*](./03-Request-Headers/Sec-Fetch.md)

## 04 — Response Headers
- [Content-Type](./04-Response-Headers/Content-Type.md)
- [Content-Length](./04-Response-Headers/Content-Length.md)
- [Content-Disposition](./04-Response-Headers/Content-Disposition.md)
- [Location](./04-Response-Headers/Location.md)
- [Server](./04-Response-Headers/Server.md)
- [Date](./04-Response-Headers/Date.md)
- [Retry-After](./04-Response-Headers/Retry-After.md)
- [Allow](./04-Response-Headers/Allow.md)
- [Accept-Ranges](./04-Response-Headers/Accept-Ranges.md)

## 05 — Security Headers
- [Strict-Transport-Security](./05-Security-Headers/Strict-Transport-Security.md)
- [Content-Security-Policy](./05-Security-Headers/Content-Security-Policy.md)
- [X-Content-Type-Options](./05-Security-Headers/X-Content-Type-Options.md)
- [X-Frame-Options](./05-Security-Headers/X-Frame-Options.md)
- [Referrer-Policy](./05-Security-Headers/Referrer-Policy.md)
- [Permissions-Policy](./05-Security-Headers/Permissions-Policy.md)
- [Cross-Origin-Opener-Policy](./05-Security-Headers/Cross-Origin-Opener-Policy.md)
- [Cross-Origin-Embedder-Policy](./05-Security-Headers/Cross-Origin-Embedder-Policy.md)
- [Cross-Origin-Resource-Policy](./05-Security-Headers/Cross-Origin-Resource-Policy.md)
- [X-DNS-Prefetch-Control](./05-Security-Headers/X-DNS-Prefetch-Control.md)

## 06 — Caching Headers
- [Cache-Control](./06-Caching-Headers/Cache-Control.md)
- [Expires](./06-Caching-Headers/Expires.md)
- [ETag](./06-Caching-Headers/ETag.md)
- [Last-Modified](./06-Caching-Headers/Last-Modified.md)
- [Age](./06-Caching-Headers/Age.md)
- [Vary](./06-Caching-Headers/Vary.md)

## 07 — CORS
- [CORS Overview](./07-CORS/CORS-Overview.md)
- [Access-Control-Allow-Origin](./07-CORS/Access-Control-Allow-Origin.md)
- [Access-Control-Allow-Methods](./07-CORS/Access-Control-Allow-Methods.md)
- [Access-Control-Allow-Headers](./07-CORS/Access-Control-Allow-Headers.md)
- [Access-Control-Allow-Credentials](./07-CORS/Access-Control-Allow-Credentials.md)
- [Access-Control-Expose-Headers](./07-CORS/Access-Control-Expose-Headers.md)
- [Access-Control-Max-Age](./07-CORS/Access-Control-Max-Age.md)
- [Access-Control-Request-Method](./07-CORS/Access-Control-Request-Method.md)
- [Access-Control-Request-Headers](./07-CORS/Access-Control-Request-Headers.md)

## 08 — Cookies
- [Cookies Overview](./08-Cookies/Cookies-Overview.md)
- [Set-Cookie](./08-Cookies/Set-Cookie.md)
- [Cookie](./08-Cookies/Cookie.md)

## 09 — Authentication
- [Authentication Overview](./09-Authentication/Authentication-Overview.md)
- [Authorization](./09-Authentication/Authorization.md)
- [WWW-Authenticate](./09-Authentication/WWW-Authenticate.md)

## 10 — Compression
- [Accept-Encoding](./10-Compression/Accept-Encoding.md)
- [Content-Encoding](./10-Compression/Content-Encoding.md)
- [Transfer-Encoding](./10-Compression/Transfer-Encoding.md)

## 11 — Content Negotiation
- [Content Negotiation Overview](./11-Content-Negotiation/Content-Negotiation-Overview.md)

## 12 — Conditional Requests
- [Conditional Requests Overview](./12-Conditional-Requests/Conditional-Requests-Overview.md)
- [If-None-Match](./12-Conditional-Requests/If-None-Match.md)
- [If-Modified-Since](./12-Conditional-Requests/If-Modified-Since.md)

## 13 — Range Requests
- [Range Requests Overview](./13-Range-Requests/Range-Requests-Overview.md)
- [Range](./13-Range-Requests/Range.md)
- [Content-Range](./13-Range-Requests/Content-Range.md)

## 14 — Proxies
- [Proxies Overview](./14-Proxies/Proxies-Overview.md)
- [Forwarded](./14-Proxies/Forwarded.md)
- [X-Forwarded-For](./14-Proxies/X-Forwarded-For.md)
- [X-Forwarded-Proto](./14-Proxies/X-Forwarded-Proto.md)
- [Via](./14-Proxies/Via.md)

## 15 — CDNs
- [CDN Caching Overview](./15-CDNs/CDN-Caching-Overview.md)

## 16 — Reverse Proxies
- [Reverse Proxy Overview](./16-Reverse-Proxies/Reverse-Proxy-Overview.md)

## 17 — Express.js
- [Headers in Express](./17-ExpressJS/Headers-in-Express.md)

## 18 — React
- [Headers and React](./18-React/Headers-and-React.md)

## 19 — Debugging
- [Debugging Headers](./19-Debugging/Debugging-Headers.md)

## 20 — Real-World Architectures
- [End-to-End Header Flow](./20-Real-World-Architectures/End-to-End-Header-Flow.md)

## 21 — Best Practices
- [Best Practices](./21-Best-Practices/Best-Practices.md)

## 22 — Reference
- [Cheat Sheet](./22-Reference/Cheat-Sheet.md)
- [Glossary](./22-Reference/Glossary.md)
