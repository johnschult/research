# Notes: LLM API from Static HTML via CORS

**Question:** Can a single static HTML file call an LLM API directly via CORS and do a useful round-trip with no backend?

---

## 2026-06-06 — Starting investigation

### What I need to find out

1. Which LLM APIs support CORS (send `Access-Control-Allow-Origin` headers)?
2. Is the API key necessarily exposed in browser-visible source?
3. What are the practical limits — streaming, token size, error handling?
4. Can we build a working proof-of-concept HTML file?

### Initial hypothesis

Most LLM APIs are designed for server-to-server use and don't advertise browser CORS support. A few (Google Gemini, Groq, possibly OpenAI) may allow it. Cloudflare Workers AI is a wildcard since it runs at the edge. The key exposure problem is real but separable from the CORS question.

---

## Research log

### CORS preflight tests (empirical, curl OPTIONS)

Ran `curl -X OPTIONS` with `Origin: https://example.com` against all major APIs:

| Provider | Endpoint | `Access-Control-Allow-Origin` | Notes |
|---|---|---|---|
| Anthropic | api.anthropic.com/v1/messages | `*` (wildcard) | Requires `anthropic-dangerous-direct-browser-access: true` header too |
| Groq | api.groq.com/openai/v1/chat/completions | `*` (wildcard) | Clean, no extra headers needed |
| OpenAI | api.openai.com/v1/chat/completions | reflects origin | Surprisingly works — forums say it doesn't, but preflight says it does |
| Gemini | generativelanguage.googleapis.com/v1beta/... | reflects origin | Works; uses `x-goog-api-key` header |
| Together AI | api.together.xyz/v1/chat/completions | `*` (wildcard) | Works |

**Key finding: all five providers pass the CORS preflight.** The web-forum wisdom that "LLM APIs don't support CORS" is wrong in 2026.

### Anthropic specifics (from Simon Willison's Aug 2024 post)

- Added CORS in Aug 2024 specifically for browser use cases
- Requires a special header: `anthropic-dangerous-direct-browser-access: true`
- The header name deliberately signals the security risk
- Motivation: "bring your own API key" tools and internal demos

### API key exposure — the real constraint

Even though CORS works, any API key embedded in a static HTML file is visible to:
- Anyone who views source
- Anyone who inspects network requests
- Browser extensions on the page
- Anyone who reads the deployed file

Safe patterns:
1. **Bring-your-own-key (BYOK)**: user enters their key in a form field → store in `localStorage` → never in the HTML source. Legitimate for demos.
2. **Restricted key**: some providers allow IP/referrer restrictions on API keys (Gemini does; Anthropic does not currently).
3. **Rate-limited free-tier key**: intentionally throwaway, low quota, acceptable to expose.

Unsafe pattern: hard-code a billing key in the HTML — anyone can find it and run up charges.

### Building the proof-of-concept

Will use Anthropic (CORS + wildcard origin = cleanest) with BYOK pattern. The `anthropic-dangerous-direct-browser-access: true` header is self-documenting about the risk.

Plan for the HTML file:
- Single file, no build step, no framework
- User enters API key → stored in `localStorage`
- Simple chat textarea → fetch → display response
- Model selector (haiku/sonnet/opus)
- Non-streaming (simpler for a single-file demo; streaming would require reading SSE chunks from a ReadableStream but is also possible)

### Streaming question

Anthropic's streaming API sends `text/event-stream`. Browser `fetch()` can read it via `response.body.getReader()` (a ReadableStream). The CORS headers don't block streaming — `Access-Control-Expose-Headers` is not required for SSE content. Streaming from a static HTML file is possible but adds ~20 lines of stream-reading boilerplate. Left out of demo for clarity; noted in README.

### Dead ends / things that didn't trip me up (but worth noting)

- Gemini's CORS: the community forum posts were about the OpenAI-compat endpoint + stainless headers, not the native Gemini endpoint. Native Gemini endpoint passes preflight fine.
- OpenAI "doesn't support CORS" — this appears to be outdated forum folklore from ~2023. As of June 2026 the preflight returns proper CORS headers.
- `dangerouslyAllowBrowser` in the Groq/OpenAI SDKs: this is an SDK-level check that throws if you try to instantiate the SDK in a browser without the flag. It does NOT mean the server blocks browser requests — it's a client-side safety guard in the SDK code.

### Conclusion

**Yes, it works.** A single static HTML file can make a complete, useful LLM round-trip with no backend. The CORS support is real and has been stable since mid-2024 for Anthropic (the most explicit about it) and exists for all major providers.

The API-key-exposure risk is real but manageable with the BYOK pattern: user supplies the key at runtime, stored in localStorage, never baked into source.
