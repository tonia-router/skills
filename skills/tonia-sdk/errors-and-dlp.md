# Errors, streaming, and DLP

See also [`tonia-api`](https://github.com/tonia-router/tonia-api).

## Hard errors

Most runtime errors use:

```json
{"error": {"type": "...", "code": "...", "retryable": false}}
```

Raise a typed exception per `type`. The SDK does not auto-retry. Honor
`retryable` and `Retry-After` (seconds) when present. Public model 404
uses a flat `{"error": "not_found"}` body.

| `type` | SDK class | Typical cause |
| --- | --- | --- |
| `authentication_error` | `AuthenticationError` | Missing / revoked `tonia_*` key (`missing_bearer`) |
| `billing_error` | `BillingError` | Past due / suspended / cancelled |
| `byok_key_missing` | `ByokKeyMissingError` | BYOK provider key not loaded |
| `tenant_upstream_blocked` | `TenantUpstreamBlockedError` | Workspace blocked that upstream |
| `invalid_request_error` | `InvalidRequestError` | Bad body or wrong surface (see Gemini below) |
| `client_error` | `PathNotAllowedError` | `client.request` path outside the allowlist |

Catch `PolicyBlockError`, `RateLimitError`, and `EntitlementError` on
runtime calls. Do not treat every 429 as admission.

### Admission 429 (`RateLimitError`)

Pass refuses the call immediately. It is not queued.

```json
{
  "error": {
    "type": "rate_limit_error",
    "code": "admission_rate_limited",
    "reason": "rpm_per_key",
    "scope": "key",
    "retryable": true
  }
}
```

Header: `Retry-After: <seconds>`. Official SDKs expose this as
`retryAfterSeconds` (TypeScript) / `retry_after_seconds` (Python).

| `reason` | `scope` | Meaning |
| --- | --- | --- |
| `rpm_per_key` | `key` | This key sent too many requests in the last minute |
| `concurrency_per_key` | `key` | This key has too many calls in flight |
| `concurrency_per_tenant` | `tenant` | This workspace has too many calls in flight |
| `concurrency_global` | `global` | Platform-wide in-flight cap |

Per-key RPM defaults to 600. In-flight concurrency is a separate limit
from RPM. A streaming call holds a slot until the stream ends.

### Do not mix these 429 / 402 / 503 families

| HTTP (hard-error mode) | `type` | `code` | SDK class | `retryable` | `Retry-After` |
| --- | --- | --- | --- | --- | --- |
| 429 | `rate_limit_error` | `admission_rate_limited` | `RateLimitError` | true | seconds (RPM: remaining window; concurrency: typically 1) |
| 429 | `entitlement_error` | `request_quota_exhausted` | `EntitlementError` | true | seconds until monthly reset |
| 402 | `entitlement_error` | `*_budget_exhausted` | `EntitlementError` | false | none — do not retry |
| 503 | `api_error` | `audit_tip_contention` | `ApiError` | true | 1 |
| 503 | `managed_credential_unavailable` | `managed_credential_unavailable` | `ManagedCredentialUnavailableError` | true | 60 |

On chat-like routes, quota and budget often arrive as HTTP 200 with
`_tonia_entitlement_block` instead of 429/402. The SDK still raises
`EntitlementError`. Branch on class + `code`, not status alone.

Upstream or other audit 502/503 uses `type: api_error` (`ApiError`).

## Carriers on HTTP 200

On chat-like routes, Pass may still return HTTP 200 with
`_tonia_policy_block` or `_tonia_entitlement_block`. Official SDKs raise
`PolicyBlockError` / `EntitlementError` before returning. Catch those
classes — do not inspect the raw body for those keys yourself.

## Streaming blocks

Pass decides policy/entitlement **before** the upstream stream. A blocked
SSE response is HTTP 200 with `x-tonia-policy-block` or
`x-tonia-entitlement-block`, plus a short synthetic body.

The SDK raises `PolicyBlockError` / `EntitlementError` from those headers
before yielding any SSE events. Body carriers remain a fallback
(Anthropic nested `message_start.message`, or a later OpenAI chunk).

## Soft-limit headers

Successful calls may include `x-tonia-limit-*` headers when usage is ≥ 80% of
a quota or budget. Below that, `lastLimits` / `last_limits` is empty — that
is expected. These are warnings, not errors. Read them from
`client.lastLimits` (TypeScript) or `client.last_limits` (Python) after a
call. A SaaS recipe is cookbook `09-saas-integrator`.

## Content redaction

Redact mode is configured in the [tonia portal](https://portal.tonia.ca) on
**Policies → Profiles** by binding a key to a redact-mode profile. SDKs do
not set a redact header and do not mask content client-side. The key stores
`profile_id` only; profile edits (and Workspace Policies that merge keywords
or fill omitted detection) apply on the next request.

`PolicyBlockError` is the runtime denial type (`error.type: policy_block`).
That is not a Policy entity — Policies / Politiques is the Workspace settings
page. Warn-mode profiles still forward; the SDK sees a normal 200.

Recommended workspaces default to Precise (`soft`) detection. A bare
identifier string may not trip a block.

## Image inputs

Send image bytes inline as a base64 data URL. Pass does not fetch remote
`http://` or `https://` image references because MediaGuard cannot inspect
their bytes before the provider receives them.

A structured remote image reference returns:

```json
{
  "error": {
    "type": "invalid_request_error",
    "code": "remote_image_url_not_supported",
    "retryable": false
  }
}
```

Do not retry the same URL. Download the image in the client application,
validate its size and type, then send it as
`data:<mime-type>;base64,<encoded-bytes>`. Never disable TLS verification or
ask Pass to fetch private/internal URLs.

## Gemini image routing

`client.images.generate` / `client.images.edit` call `/v1/images/*`
for openai, xAI, and StepFun. Gemini image SKUs must use
`client.interactions.create` (`POST /v1/interactions`).

A Gemini SKU on `/v1/images/*` returns:

```json
{
  "error": {
    "type": "invalid_request_error",
    "code": "provider_requires_surface",
    "required_surface": "interactions",
    "retryable": false
  }
}
```

That maps to `InvalidRequestError`. Do not retry on `/v1/images/*`. Call
`/v1/interactions` instead. Gemini Interactions **edit** parts are
`{type: image, mime_type, data}` with raw base64 — not a data URL.
