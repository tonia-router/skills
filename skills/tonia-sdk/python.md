# Python — `tonia`

Need a key first? [Portal key setup](setup.md). Pointing Cursor at tonia?
[Coding tools](tools.md).

Read [`compatibility.json`](compatibility.json) before installing.

```bash
# packages are not on PyPI yet — editable install from the sibling tree
pip install -e ../python-sdk
# or: pip install -e /path/to/tonia-router/python-sdk

# once published: pip install tonia
```

Needs Python 3.11 or newer (3.12–3.14 are current). Python 3.10 reaches
end of support in October 2026 — do not target it.

The client import is `from tonia import Tonia`. The Pass server also ships a
Python package named `tonia` — install this SDK from
`tonia-router/python-sdk` so `import tonia` is the client.

```python
import os
from tonia import AsyncTonia, EntitlementError, PolicyBlockError, RateLimitError, Tonia

with Tonia(api_key=os.environ["TONIA_API_KEY"]) as client:
    client.status.get()
    client.catalogue.list()
    public_models = client.public_models.list()
    client.public_model_categories.list()
    public_ids = [model["id"] for model in public_models["data"]]
    if public_ids:
        client.public_models.get(public_ids[0])

    listed = client.models.list()
    ids = [model["id"] for model in listed["data"]]
    if not ids:
        raise RuntimeError("this key has no models; check the profile allowlist in the portal")

    # models.list() is Bearer / OpenAI-shaped (anthropic/claude-…).
    # messages.create sends x-api-key but still takes that same id.

    def pick(test):
        return next((model_id for model_id in ids if test(model_id)), None)

    client.models.get(ids[0])

    client.chat.completions.create(
        model=ids[0],
        messages=[{"role": "user", "content": "Bonjour"}],
    )

    for event in client.chat.completions.stream(
        model=ids[0],
        messages=[{"role": "user", "content": "Bonjour"}],
    ):
        pass  # event.json is a provider-shaped chunk when present

    client.responses.create(model=ids[0], input="Bonjour")

    anthropic = pick(lambda model_id: model_id.startswith("anthropic/"))
    if anthropic:
        # sends x-api-key automatically
        client.messages.create(
            model=anthropic,
            max_tokens=256,
            messages=[{"role": "user", "content": "Bonjour"}],
        )

    embedding = pick(lambda model_id: "embedding" in model_id)
    if embedding:
        client.embeddings.create(model=embedding, input="Bonjour")

    rerank = pick(lambda model_id: "rerank" in model_id)
    if rerank:
        client.rerank.create(model=rerank, query="q", documents=["a"])

    image_sku = pick(
        lambda model_id: "turbo" not in model_id.lower()
        and not model_id.startswith(("gemini/", "gemini-"))
        and (
            model_id.startswith(("openai/", "xai/"))
            or model_id.lower().startswith(("gpt-image", "grok-imagine-image", "muse-image"))
        )
    )
    if image_sku:
        # images.generate — 300s unless timeout= is set on Tonia(...)
        # Tenant dial (`1k`/`2k`/`4k`) or OpenAI size. Do not send resolution.
        client.images.generate(
            model=image_sku,
            prompt="Draw a red fox",
            n=1,
            size="2k",
        )

    gemini_image = pick(
        lambda model_id: model_id.startswith("gemini/") and "image" in model_id.lower()
    )
    if gemini_image:
        # Gemini image SKUs — /v1/interactions (not images.generate)
        client.interactions.create(
            model=gemini_image,
            input="Draw a red fox",
            stream=False,
        )

    tts = pick(
        lambda model_id: (
            "tts" in model_id.lower()
            and "realtime" not in model_id.lower()
            and "gemini" not in model_id.lower()
        )
    )
    if tts:
        client.audio.speech.create(model=tts, input="Bonjour", voice="alloy")

    stt = pick(
        lambda model_id: (
            (
                "transcribe" in model_id.lower()
                or "whisper" in model_id.lower()
                or "asr" in model_id.lower()
            )
            and "tts" not in model_id.lower()
            and "realtime" not in model_id.lower()
            and "gemini" not in model_id.lower()
        )
    )
    if stt:
        client.audio.transcriptions.create(
            model=stt,
            file=open("clip.wav", "rb").read(),
            filename="clip.wav",
        )

    gemini_token_audio = pick(
        lambda model_id: model_id.startswith("gemini/")
        and ("tts" in model_id.lower() or "transcribe" in model_id.lower())
    )
    if gemini_token_audio:
        # Gemini token TTS/STT — never audio.speech / audio.transcriptions
        client.interactions.create(
            model=gemini_token_audio,
            input="Bonjour",
            stream=False,
        )

    try:
        client.chat.completions.create(
            model=ids[0],
            messages=[{"role": "user", "content": "Bonjour"}],
        )
    except PolicyBlockError:
        pass  # Bind a redact-mode profile in the portal, then retry.
    except RateLimitError as err:
        if err.retryable:
            wait = err.retry_after_seconds or 1
            # sleep `wait` seconds, then retry once
        else:
            raise
    except EntitlementError as err:
        # quota: retryable after Retry-After. budget (*_budget_exhausted): do not retry
        if not err.retryable:
            raise

    client.last_limits

async with AsyncTonia(api_key=os.environ["TONIA_API_KEY"]) as client:
    await client.models.list()
```

Chat helpers default to a 60s timeout. Image / interactions helpers default
to 300s unless you pass `timeout=` on `Tonia(...)`. Prefer `.stream()` over
`create(stream=True)`.

The SDK does not auto-retry. See [Errors and DLP](errors-and-dlp.md).
LLM `tools` are a passthrough body field — [Coding tools](tools.md).

Confirm signatures against the installed package and
[`tonia-api`](https://github.com/tonia-router/tonia-api).

SaaS apps store their own `messages[]`, meter provider `usage`, and read
`last_limits` (only when usage is ≥ 80%). Billing and keys stay in the
[portal](https://portal.tonia.ca). Cookbook:
[`sdk-examples`](https://github.com/tonia-router/sdk-examples) `09_saas_integrator`.

Gemini image generate uses a string `input`. Edit uses
`[{type: "text", "text": "..."}, {type: "image", "mime_type": "image/png", "data": raw_b64}]`
(`data` is raw base64). Parse `model_output` image parts from the native
response. See the Python SDK README **Images** section.
