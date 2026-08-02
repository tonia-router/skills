---
name: tonia-sdk
description: >-
  Install, configure, call, debug, or migrate to the official tonia Pass
  SDKs (@tonia/sdk, tonia, tonia-sdk). Use when working with pass.tonia.ca,
  TONIA_API_KEY, catalogue/models/conversations, streaming, or tonia
  policy and entitlement errors.
---

# tonia-sdk

Guidance for using the official tonia Pass SDKs from coding agents.

## Safe workflow

1. Detect the repository language and package manager before installing.
2. Read `TONIA_API_KEY` from the environment — never print, commit, log, or
   paste live keys.
3. Default base URL: `https://pass.tonia.ca`. Override only for an explicit
   local or on-prem target. Never call a model provider directly.
4. Prefer typed SDK resources. Use the raw request helper only for supported
   Pass path prefixes listed in the SDK docs.
5. Handle streaming without buffering. On success responses, still check for
   `_tonia_policy_block` or `_tonia_entitlement_block` and treat those as
   errors.
6. Content redaction is configured in the [tonia portal](https://portal.tonia.ca)
   by binding a key to a redact-mode profile — not via an SDK header.

## Language guides

- [TypeScript](typescript.md) — `@tonia/sdk`
- [Python](python.md) — `tonia`
- [Rust](rust.md) — `tonia-sdk`
- [Errors and DLP](errors-and-dlp.md)
