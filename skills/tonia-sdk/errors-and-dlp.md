# Errors, streaming, and DLP

See also [`tonia-api`](https://github.com/tonia-router/tonia-api)
(`public.openapi.yaml`).

## Hard errors

Most runtime errors use:

```json
{"error": {"type": "...", "code": "...", "retryable": false}}
```

Raise a typed exception per `type`. Honor `retryable` and `Retry-After`
when present. Conversation routes omit `retryable`. Public model 404 uses
a flat `{"error": "not_found"}` body.

## Carriers on HTTP 200

On chat-like routes, an HTTP 200 body may still include:

- `_tonia_policy_block` — treat as a policy error (no `retryable`)
- `_tonia_entitlement_block` — treat as an entitlement error

Do not treat those responses as successful completions.

## Soft-limit headers

Successful calls may include `x-tonia-limit-*` headers when usage is near a
quota or budget. These are warnings, not errors. Read them from
`client.lastLimits` (TypeScript) or `client.last_limits` (Python) after a
call.

## Content redaction

Redact mode is configured in the [tonia portal](https://portal.tonia.ca) by
binding a key to a redact-mode profile. SDKs do not set a redact header and
do not mask content client-side.
