---
name: tonia-sdk
description: >-
  Set up tonia from a portal API key through Cursor, Claude Code, Codex, or
  the official SDKs (@tonia-router/sdk, tonia). Use when creating a tonia_sk_ key,
  setting TONIA_API_KEY, pointing a coding tool at pass.tonia.ca, installing
  the SDK, passing tools, streaming, or handling policy_block, RateLimitError,
  Retry-After, or entitlement errors.
---

# tonia-sdk

Walk the user from a Workspace key to a working call. Do not mint keys in
code. Stay on the public SDK surface.

## Pick a path

1. **No `TONIA_API_KEY` yet** — read [Portal key setup](setup.md). Stop until
   the key is in the environment. Never print, commit, log, or paste the
   bearer.
2. **Point an existing tool at tonia** (Cursor, Claude Code, Codex, …) —
   read [Coding tools](tools.md). Do not install `@tonia-router/sdk` into those
   tools; they want a base URL + key only.
3. **Write app code with the official SDK** — read
   [`compatibility.json`](compatibility.json) first. Detect the repo
   language, then read [TypeScript](typescript.md) or [Python](python.md).
   Rust is not shipped; see [Rust](rust.md).
4. **Errors, streaming, DLP, images** — [Errors and DLP](errors-and-dlp.md).
5. Confirm signatures against the installed package and
   [`compatibility.json`](compatibility.json).

## Safe workflow

1. Detect the repository language and package manager before installing.
   If `compatibility.json` `status` is `unpublished`, install from the
   local SDK tree — do not `npm add` / `pip install` from the registry.
   If `status` is `planned`, skip that package.
   Follow `runtimes` in that file: Python `>=3.11`, Node.js `>=22`. Do not
   target Python 3.10 or Node 18/20 (end-of-life).
2. Default base URL: `https://pass.tonia.ca:8443`. Override only for an
   explicit local or on-prem target. Never call a model provider directly.
3. Prefer typed SDK helpers (`catalogue.list`, `models.list`,
   `chat.completions.create`, `responses.create`, …). Use `client.request`
   only for a supported Pass path that has no named helper.
4. Call `client.models.list()` before picking a model — that list is what
   this key may call (bound profile, resolved live). Empty list → stop.
   Do not hardcode a SKU the list does not contain. Skip a helper when no
   listed id matches that surface (chat, embeddings, `/v1/images`, Gemini
   image, …). `models.list()` is Bearer / OpenAI-shaped (`anthropic/claude-…`).
   Anthropic clients listing with `x-api-key` see unprefixed ids (`claude-…`).
   Do not mix the two styles.
5. Use `.stream()` for SSE. Do not buffer the stream. The SDK already raises
   `PolicyBlockError` / `EntitlementError` on HTTP 200 carriers — catch the
   typed class; do not re-parse `_tonia_policy_block` yourself.
6. Content redaction is configured in the [tonia portal](https://portal.tonia.ca)
   (Policies → Profiles), not via an SDK header.
7. LLM function tools are an untyped passthrough body field (`tools`). The
   SDK has no agent loop. See [Coding tools](tools.md).
8. The SDK does not auto-retry. Follow `retryable` and `Retry-After`.
   Admission 429 is `RateLimitError`. Monthly quota 429 is `EntitlementError`.
   Budget exhaustion is not retryable.

## Language guides

- [Portal key setup](setup.md)
- [Coding tools](tools.md)
- [TypeScript](typescript.md) — `@tonia-router/sdk`
- [Python](python.md) — `tonia`
- [Rust](rust.md) — not shipped
- [Errors and DLP](errors-and-dlp.md)
- SaaS integrator (your history, `usage`, limits) —
  [`sdk-examples`](https://github.com/tonia-router/sdk-examples) `09-saas-integrator`
