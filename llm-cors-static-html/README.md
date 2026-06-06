# Can a static HTML file call an LLM API directly?

**Question:** Can a single static HTML file call an LLM API directly via CORS and complete a useful round-trip — no backend, no build step, no proxy?

**Short answer: Yes.** All five major providers tested pass the CORS preflight as of June 2026. The technique works and is practical for demos, internal tools, and "bring your own key" apps.

---

## What I did

1. Ran `curl -X OPTIONS` preflight requests against five LLM API endpoints, spoofing `Origin: https://example.com`, to get ground-truth CORS headers.
2. Researched Anthropic's August 2024 blog post announcing browser CORS support and the required opt-in header.
3. Built a single-file HTML demo (`demo.html`) that calls `api.anthropic.com/v1/messages` directly from the browser using `fetch()`, with no backend of any kind.

---

## CORS support by provider (June 2026)

Tested empirically with an `OPTIONS` preflight. All passed.

| Provider | Endpoint | `Access-Control-Allow-Origin` | Extra requirement |
|---|---|---|---|
| **Anthropic** | `api.anthropic.com/v1/messages` | `*` | Must send `anthropic-dangerous-direct-browser-access: true` header |
| **Groq** | `api.groq.com/openai/v1/chat/completions` | `*` | None |
| **OpenAI** | `api.openai.com/v1/chat/completions` | reflects origin | None |
| **Gemini** | `generativelanguage.googleapis.com/v1beta/...` | reflects origin | Use `x-goog-api-key` (not `Authorization`) |
| **Together AI** | `api.together.xyz/v1/chat/completions` | `*` | None |

> **Note:** The widespread forum belief that OpenAI and Gemini block browser requests appears to be outdated (circa 2023). As of June 2026 both pass the preflight. Some older confusion stems from SDK-level guards (`dangerouslyAllowBrowser: true` in the JS SDKs), which are client-side safety checks in the library code — not server-side blocks.

---

## The API key exposure problem

This is the real constraint — not CORS.

Any API key **embedded in a static HTML file** is visible to:
- Anyone who views the page source
- Anyone who opens DevTools → Network tab
- Browser extensions running on the page
- Anyone who reads the deployed file on disk or a CDN

**Safe patterns:**

| Pattern | How | When appropriate |
|---|---|---|
| **Bring your own key (BYOK)** | User pastes key into a form; stored in `localStorage` | Internal tools, local experiments, demos where each user has their own key |
| **Restricted API key** | Provider restricts key to specific HTTP referrer or IP | Gemini supports this; Anthropic does not currently |
| **Throwaway key** | Low quota, rate-limited free-tier key you deliberately expose | Public demos with acceptable loss if key is scraped |

**Unsafe pattern:** Hard-coding a billing key in a static file you deploy publicly. Anyone can extract it and run up charges.

The demo (`demo.html`) uses BYOK: the user enters their key at runtime, it is stored only in the browser's `localStorage`, and it is never baked into the HTML source.

---

## The working demo

`demo.html` is a self-contained file (~200 lines, no dependencies) that:

1. Prompts the user for an Anthropic API key and stores it in `localStorage`
2. Lets the user pick a model (Haiku / Sonnet / Opus) and type a prompt
3. Calls `https://api.anthropic.com/v1/messages` directly via `fetch()`
4. Displays the response text and token usage metadata

To run it: open `demo.html` in any browser. No server needed.

The critical fetch headers:

```js
fetch('https://api.anthropic.com/v1/messages', {
  method: 'POST',
  headers: {
    'x-api-key': apiKey,
    'anthropic-version': '2023-06-01',
    'content-type': 'application/json',
    'anthropic-dangerous-direct-browser-access': 'true',   // ← required opt-in
  },
  body: JSON.stringify({ model, max_tokens: 1024, messages: [...] }),
})
```

---

## What I did not implement (and why)

**Streaming:** Anthropic's streaming endpoint sends `text/event-stream`. A browser `fetch()` can read it via `response.body.getReader()` (a `ReadableStream`). CORS does not block streaming. It is feasible in a static HTML file but adds ~20 lines of stream-parsing boilerplate. Omitted from the demo to keep it legible.

**Multi-turn conversation:** Requires maintaining a `messages[]` array across turns. Straightforward to add; out of scope for a proof of concept.

---

## Conclusion

A single static HTML file can call an LLM API directly via CORS and complete a useful round-trip with no backend. The answer is **yes with one important caveat:**

> **The CORS constraint is solved. The API key exposure constraint is not — it requires a deliberate pattern choice (BYOK, restricted key, or throwaway key).** For anything beyond local experiments or internal tools, you still need a backend or a key-restriction mechanism.

The technique is legitimate and production-viable for internal tools where users supply their own keys. It is not viable for public-facing products where a shared API key is needed.

---

*Investigation date: 2026-06-06*
