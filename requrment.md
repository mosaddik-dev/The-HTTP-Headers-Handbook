# Claude Code Prompt

You are an expert web platform engineer, senior backend engineer, technical educator, and technical book author.

Your task is to create a complete production-grade handbook about **HTTP Headers**.

This is **not** a quick explanation.

This should become my permanent reference manual that completely replaces reading MDN for day-to-day development.

Assume the reader is an intermediate to advanced full-stack JavaScript developer who already knows:

- JavaScript
- TypeScript
- React
- Node.js
- Express.js
- REST APIs
- Authentication
- Basic networking

Do **not** explain JavaScript basics.

Instead, explain HTTP headers deeply from the perspective of someone building production web applications.

---

# Primary Goal

Create the most complete HTTP Headers handbook possible.

The handbook should explain:

- what every important HTTP header is
- why it exists
- what problem it solves
- when to use it
- when NOT to use it
- how browsers use it
- how servers use it
- how proxies use it
- how CDNs use it
- how reverse proxies use it
- how Node.js and Express interact with it
- how React applications are affected by it
- common mistakes
- production best practices
- security implications
- debugging techniques

This handbook should cover **100% of the production HTTP headers** that developers actually encounter.

Do **NOT** include obsolete or experimental headers unless they are necessary for understanding production systems.

---

# The handbook should feel like a book

Do NOT write one huge markdown file.

Create a structured documentation project.

Example structure:

/HTTP-Headers-Handbook

README.md

Progress.md

SUMMARY.md

01-Introduction/

- What are HTTP Headers.md
- Anatomy of an HTTP Message.md
- Request vs Response Headers.md
- End-to-End vs Hop-by-Hop Headers.md
- How Browsers Process Headers.md
- How Servers Process Headers.md
- HTTP Versions and Headers.md

02-Core-Concepts/

03-Request-Headers/

04-Response-Headers/

05-Security-Headers/

06-Caching-Headers/

07-CORS/

08-Cookies/

09-Authentication/

10-Compression/

11-Content-Negotiation/

12-Conditional-Requests/

13-Range-Requests/

14-Proxies/

15-CDNs/

16-Reverse-Proxies/

17-ExpressJS/

18-React/

19-Debugging/

20-Real-World-Architectures/

21-Best-Practices/

22-Reference/

Every HTTP header should have its own markdown file.

For example:

Accept.md

Authorization.md

Content-Type.md

Content-Length.md

Host.md

Origin.md

Referer.md

Referrer-Policy.md

Cache-Control.md

ETag.md

If-None-Match.md

If-Modified-Since.md

Set-Cookie.md

Cookie.md

Accept-Encoding.md

Content-Encoding.md

Transfer-Encoding.md

Connection.md

Upgrade.md

Location.md

Vary.md

Accept-Language.md

User-Agent.md

Server.md

Date.md

Age.md

Expires.md

Last-Modified.md

Range.md

Content-Range.md

WWW-Authenticate.md

Retry-After.md

X-Forwarded-For.md

X-Forwarded-Proto.md

Forwarded.md

Strict-Transport-Security.md

Content-Security-Policy.md

Permissions-Policy.md

Cross-Origin-Opener-Policy.md

Cross-Origin-Embedder-Policy.md

Cross-Origin-Resource-Policy.md

X-Content-Type-Options.md

X-Frame-Options.md

X-DNS-Prefetch-Control.md

Access-Control-Allow-Origin.md

Access-Control-Allow-Headers.md

Access-Control-Allow-Methods.md

Access-Control-Allow-Credentials.md

Access-Control-Expose-Headers.md

Access-Control-Max-Age.md

Access-Control-Request-Headers.md

Access-Control-Request-Method.md

Origin.md

Referer.md

and every other commonly used production header.

---

# Every header page MUST follow exactly this structure

# Header Name

## Quick Summary

One paragraph.

## What problem does this header solve?

Explain the real-world problem.

## Why was it introduced?

Historical context.

## How does it work?

Explain browser behavior.

Explain server behavior.

Explain proxy behavior.

Explain CDN behavior.

Explain reverse proxy behavior.

## HTTP Request Example

Show raw HTTP.

## HTTP Response Example

Show raw HTTP.

## Express.js Example

Use production-ready Express code.

Explain every line.

## Node.js Example

If applicable.

## React Example

If applicable.

Explain how React indirectly depends on this header.

If React does not interact with it directly, explain why.

## Browser Lifecycle

Explain exactly what the browser does.

## Production Use Cases

Several real examples.

## Common Mistakes

Examples.

## Security Considerations

Examples.

## Performance Considerations

Examples.

## Reverse Proxy Considerations

Examples using Nginx or similar.

## CDN Considerations

Examples.

## Cloud Deployment Considerations

Examples.

## Debugging

How to inspect it using:

- Chrome DevTools
- curl
- Postman
- Bruno
- Node.js
- Express logging

## Best Practices

Checklist.

## Related Headers

Explain the relationship.

## Mental Model

Give an intuitive explanation.

---

# Code examples

This is extremely important.

Every chapter must include realistic production code.

Use examples from:

- Express.js
- Node.js
- React (only when relevant)
- Fetch API
- Axios
- Browser DevTools
- curl
- Postman
- Nginx
- Reverse proxy configurations

Every example must explain:

- what each line does
- why it exists
- what happens if removed
- production implications

Never show toy examples if a production example is possible.

---

# Also explain the surrounding technologies

The handbook should explain how HTTP headers interact with:

- HTTP
- HTTPS
- TLS
- Browsers
- React
- Express
- Node.js
- REST APIs
- GraphQL
- Cookies
- Sessions
- JWT
- OAuth
- Authentication
- Authorization
- CORS
- CSP
- Same Origin Policy
- CSRF
- XSS
- Clickjacking
- MIME Types
- Compression
- Caching
- CDNs
- Reverse Proxies
- Load Balancers
- Nginx
- Cloudflare
- Browser Cache
- Service Workers
- Fetch API
- Axios
- Streaming
- File Uploads
- File Downloads
- HTTP/1.1
- HTTP/2
- HTTP/3
- Server-Sent Events (SSE)
- WebSockets (especially the Upgrade header)
- API Gateways

Whenever a header relates to one of these technologies, explain that relationship.

---

# Explain interactions between headers

One of the most important parts of this handbook is understanding how headers work together.

Examples include (but are not limited to):

- Content-Type + Accept
- Cache-Control + ETag
- Cache-Control + Expires
- If-None-Match + ETag
- Last-Modified + If-Modified-Since
- Cookie + Set-Cookie
- Origin + CORS headers
- Authorization + WWW-Authenticate
- Content-Encoding + Accept-Encoding
- CSP + X-Frame-Options
- Referrer-Policy + Referer
- Strict-Transport-Security + HTTPS
- Vary + CDN caching
- Range + Content-Range
- Transfer-Encoding + Content-Length

Explain these relationships with diagrams where appropriate using Mermaid.

---

# Teaching Style

Teach like an experienced staff engineer mentoring another engineer.

Do not simply define terms.

Explain:

- WHY
- HOW
- WHEN
- WHY NOT
- WHAT HAPPENS INTERNALLY
- WHAT THE BROWSER DOES
- WHAT THE SERVER DOES

Avoid shallow summaries.

---

# Progress Tracker

This project will likely span multiple Claude sessions.

Create a file called:

Progress.md

This file must contain a checklist of every planned chapter and every planned header.

Example:

- [x] Introduction
- [x] HTTP Message Structure
- [x] Request Headers Overview
- [ ] Accept
- [ ] Authorization
- [ ] Cache-Control
- [ ] Content-Type
- [ ] Content-Length
- [ ] Cookie
- [ ] Set-Cookie
- [ ] ETag
- [ ] If-None-Match
- [ ] Last-Modified
- [ ] Strict-Transport-Security
- [ ] Content-Security-Policy
- [ ] Referrer-Policy
- [ ] Permissions-Policy
- [ ] Access-Control-Allow-Origin
- [ ] Access-Control-Allow-Headers
- [ ] Access-Control-Allow-Methods
- [ ] Access-Control-Allow-Credentials
- ...

Whenever a chapter or header is completed:

1. Mark it as completed.
2. Update the completion percentage.
3. Add the completion date.
4. Record what remains.
5. Never redo completed work unless requested.

At the start of every new session, read Progress.md first and continue from the first unfinished item.

---

# Quality Requirements

The handbook should prioritize depth over speed.

Each page should be detailed enough that an experienced web developer can understand not only how to use the header, but also why the HTTP specification includes it and how it behaves in production systems.

Avoid unnecessary repetition, but do not omit important implementation details.

The end result should feel like a professionally written engineering handbook rather than a collection of notes.

# Others

Add a "Decision Tree" for each header (e.g., "Should I use this header? If yes, under what conditions?").

Include a "Production Checklist" at the end of every chapter.

Add Mermaid sequence diagrams showing the browser, CDN, reverse proxy, server, and database interactions where relevant.

Include real packet captures or annotated raw HTTP requests/responses alongside Express code so you can connect the code to what actually travels over the network.

Finish with a "Cheat Sheet" section containing one-page summaries of all headers, grouped by category (Security, Caching, Authentication, CORS, Compression, Content Negotiation, etc.) for quick reference during development.

Add anything you think necessary.

