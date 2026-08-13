# Portal key setup

Issue the key in the [tonia portal](https://portal.tonia.ca). There is no SDK
method and no public API to mint, reveal, or rotate a `tonia_*` key.

## Issue

1. Sign in at [portal.tonia.ca](https://portal.tonia.ca).
2. Open **Members**. A solo Workspace still uses this page — you are the
   member.
3. Open the member the key is for.
4. Under **Access keys**, click **Issue a key**. If the portal asks for
   administrator approval, complete that prompt.
5. The key is bound to a **restriction profile** (the member’s assigned
   profile). Providers, models, and sensitive-data rules come from that
   profile, resolved live on every request. Editing the profile changes the
   next call — no rotation needed.
6. Copy the Bearer **once**. Click **I saved the key — close**. The portal
   will not show the full value again. Lists show the prefix only.

Never invent a key, never read one from git, and never echo it back in chat.

## Put it in the environment

```bash
export TONIA_API_KEY=tonia_sk_…
```

Use a placeholder in committed files (`.env.example`: `TONIA_API_KEY=tonia_sk_…`).

## What this key can call

After the env var is set, `client.models.list()` is the allowlist for this
key: Workspace roster ∩ profile `model_allowlist` / `provider_allowlist`.
Empty list → empty allowlist or missing BYOK — do not guess a model id.
Public `client.publicModels.list()` / `client.public_models.list()` is the
Managed sell catalogue, not what this key may call.

## Next

- Existing IDE / CLI (Cursor, Claude Code, Codex, …) → [Coding tools](tools.md)
- App code → [TypeScript](typescript.md) or [Python](python.md)
