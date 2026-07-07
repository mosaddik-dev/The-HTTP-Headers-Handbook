# The HTTP Headers Handbook

> A production-grade reference manual for HTTP headers, written for intermediate-to-advanced full-stack JavaScript engineers.

This handbook is designed to **completely replace day-to-day MDN lookups** for HTTP headers. It explains not just *what* each header does, but *why it exists*, *what problem it solves*, *how browsers, servers, proxies, CDNs, and reverse proxies treat it*, and *how it behaves in real production systems* built with Node.js, Express, and React.

## Who this is for

You already know JavaScript, TypeScript, React, Node.js, Express, REST APIs, authentication, and basic networking. This handbook does **not** teach those. It teaches HTTP headers deeply, from the perspective of someone shipping and operating production web applications.

## How to read this handbook

- **New to the topic?** Read Parts 1–2 (Introduction + Core Concepts) top to bottom.
- **Need a specific header?** Jump straight to its page via [SUMMARY.md](./SUMMARY.md).
- **Debugging an incident?** Go to [19-Debugging](./19-Debugging/) and the relevant category chapter.
- **In a hurry?** The [Cheat Sheet](./22-Reference/Cheat-Sheet.md) has one-page-per-category summaries.

## Structure

The handbook is organized into 22 parts:

| Part | Topic |
|------|-------|
| 01 | Introduction — what headers are, message anatomy, how browsers/servers process them |
| 02 | Core Concepts — categories, syntax, structured fields, forbidden headers |
| 03 | Request Headers — every header a client sends |
| 04 | Response Headers — every header a server returns |
| 05 | Security Headers — HSTS, CSP, COOP/COEP/CORP, framing, MIME sniffing |
| 06 | Caching Headers — Cache-Control, ETag, Expires, Age, Vary |
| 07 | CORS — the full cross-origin request/response header set |
| 08 | Cookies — Set-Cookie, Cookie, attributes, sessions |
| 09 | Authentication — Authorization, WWW-Authenticate, schemes, JWT/OAuth |
| 10 | Compression — Accept-Encoding, Content-Encoding, Transfer-Encoding |
| 11 | Content Negotiation — Accept family, Vary, Content-* |
| 12 | Conditional Requests — validators and preconditions |
| 13 | Range Requests — partial content, streaming, resumable downloads |
| 14 | Proxies — hop-by-hop, Via, Forwarded, X-Forwarded-* |
| 15 | CDNs — edge caching, Cloudflare, cache keys, surrogate headers |
| 16 | Reverse Proxies — Nginx patterns and header rewriting |
| 17 | Express.js — how Express reads and writes headers |
| 18 | React — how SPAs and SSR frameworks depend on headers |
| 19 | Debugging — DevTools, curl, Postman, Bruno, Node, logging |
| 20 | Real-World Architectures — end-to-end header flows |
| 21 | Best Practices — hardening, checklists, anti-patterns |
| 22 | Reference — cheat sheets, status codes, glossary, decision trees |

## Conventions used in every header page

Every header has its own file and follows a fixed structure: Quick Summary → Problem → History → How It Works (browser/server/proxy/CDN/reverse-proxy) → raw HTTP examples → Express/Node/React code → Browser Lifecycle → Production Use Cases → Common Mistakes → Security → Performance → Reverse Proxy → CDN → Cloud → Debugging → Best Practices → Related Headers → Decision Tree → Mental Model.

Code is always production-oriented and every line is explained.

## Progress

See [Progress.md](./Progress.md) for the live completion checklist. This project spans multiple sessions; work resumes from the first unchecked item.
