---
layout: page
title: Blind SSRF
parent: Examples
nav_order: 2
---

**Blind SSRF (Server-Side Request Forgery)** is a variant of SSRF where an application allows the **backend** to make a request to a URL controlled or influenced by the attacker, but **from the attacker’s perspective there is no direct feedback**.  
In a “classic” SSRF you can sometimes confirm impact because the response changes (errors, timeouts, different status codes, different content). In **Blind SSRF**, the response often stays the same: the front-end does not reflect what actually happened behind the scenes.

This is common in real-world environments because the backend may:
- execute the request in **asynchronous processes** (jobs/queues/workers),
- go through **proxies** or intermediate services that “normalize” errors,
- use **caches** or validations that hide the real outcome,
- or only perform part of the flow (for example, **DNS resolution** without completing an HTTP fetch).

That’s why the correct way to validate it is **Out-of-Band (OOB)**: instead of relying on the response, you point the payload to a domain you control (for example `introspector.sh`) and observe whether the backend produces **callbacks** to your infrastructure.

### What evidence do you usually see?

- **DNS callback:** the server resolved your domain. This is a strong signal that the backend attempted to reach the host (reachability), although it doesn’t always mean HTTP was completed.
- **HTTP callback:** the server made a real request to your listener. This is even stronger because you can observe **method, path, headers, timing**, and sometimes client behavior patterns.
- **Additional patterns:** some clients generate extra requests such as `/favicon.ico`, `/robots.txt`, prefetching, or validation checks. These details help fingerprint backend behavior and confirm it’s not noise.

In short: **Blind SSRF = you don’t see it in the response, you prove it via OOB callbacks.**  
Introspector makes this easier by capturing and correlating DNS/HTTP interactions, showing clear evidence (timestamps, IP, headers, paths) to support the finding.

---

### Practical tips (validation + safe exploitation)

- **Use unique identifiers per attempt:** add a distinct path or subdomain per payload to correlate callbacks and avoid cache/noise confusion.
- **Test *follow redirects*:** if the backend follows `301/302`, you can chain redirects (e.g., through an attacker-controlled intermediate endpoint) and in some cases **bypass allowlists** that only validate the initial URL.
- **Leverage auto-fetching as a signal:** if you see hits to `/favicon.ico` or `/robots.txt`, use it to fingerprint the client (prefetch/preview) and confirm the behavior is real.
- **Separate DNS-only vs HTTP:** DNS confirms reachability; HTTP confirms a real fetch. Clearly state which one you observed and what it proves.
- **Watch timing:** delayed callbacks often indicate queues/workers. This helps explain why nothing changes in the response.
- **Check consistency and repetition:** send the same payload multiple times. Consistent callbacks (same origin/pattern) make your evidence much stronger.

<div align="center">
  <pre><code>
┌──────────────────────────────┐        HTTP / DNS         ┌───────────────────────────────┐
│          Target App          │  ──────────────────────▶ │     Introspector Framework    │
│  - Web App                   │                           │  - HTTP Listener              │
│  - API Endpoint              │                           │  - DNS Server                 │
│  - XML / URL Parser          │                           │  - Payload Host               │
│  - Image / File Processor    │                           │  - Log Engine                 │
└──────────────────────────────┘                           └───────────────┬───────────────┘
                                                                           │
                                                                           │  captures & correlates
                                                                           ▼
                                                              ┌──────────────────────────────┐
                                                              │           Callbacks          │
                                                              │  - Timestamps                │
                                                              │  - Source IP / ASN           │
                                                              │  - HTTP Headers              │
                                                              │  - Request Path / Body       │
                                                              │  - Response Behavior         │
                                                              └───────────────┬──────────────┘
                                                                              │  real-time view
                                                                              ▼
┌──────────────────────────────┐        Real-time         ┌──────────────────────────────┐
│          Web UI Logs         │  ◀───────────────────── │        Evidence & Timeline   │
│  - HTTP Requests             │                          │  - Sessions / Correlation    │
│  - DNS Queries               │                          │  - Redirect / Fetch patterns │
│  - Full Headers              │                          │  - Validation of Blind SSRF  │
│  - Request Bodies            │                          │                              │
└──────────────────────────────┘                          └──────────────────────────────┘
  </code></pre>
</div>
