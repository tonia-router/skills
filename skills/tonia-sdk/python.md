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
from tonia import AgentBlockError, AsyncTonia, EntitlementError, PolicyBlockError, RateLimitError, Tonia

with Tonia(api_key=os.environ["TONIA_API_KEY"]) as client:
    client.status.get()
    client.catalogue.list()
    public_models = client.public_models.list()
    client.public_model_categories.list()
    public_ids = [model["id"] for model in public_models["data"]]
    if public_ids:
        client.public_models.get(public_ids[0])

    listed = client.models.list()
    rows = listed["data"]
    ids = [model["id"] for model in rows]
    if not ids:
        raise RuntimeError("this key has no models; check the profile allowlist in the portal")

    # models.list() is Bearer / OpenAI-shaped (anthropic/claude-…).
    # messages.create sends x-api-key but still takes that same id.

    def caps(model):
        raw = model.get("capabilities")
        return [str(cap) for cap in raw] if isinstance(raw, list) else []

    def surface_of(model):
        raw = model.get("surface")
        return raw if isinstance(raw, dict) else {}

    def pick(test):
        return next((model["id"] for model in rows if test(model)), None)

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

    anthropic = pick(lambda model: str(model["id"]).startswith("anthropic/"))
    if anthropic:
        # sends x-api-key automatically
        client.messages.create(
            model=anthropic,
            max_tokens=256,
            messages=[{"role": "user", "content": "Bonjour"}],
        )

    embedding = pick(
        lambda model: surface_of(model).get("family") == "embeddings"
        or "embeddings" in caps(model)
        or (
            "embed" in str(model["id"]).lower()
            and "rerank" not in str(model["id"]).lower()
        )
    )
    if embedding:
        # Tenant /v1/embeddings — send model + input. Not /v2/embed.
        client.embeddings.create(model=embedding, input="Bonjour")

    rerank = pick(
        lambda model: surface_of(model).get("family") == "rerank"
        or "rerank" in caps(model)
        or "rerank" in str(model["id"]).lower()
    )
    if rerank:
        # Tenant /v1/rerank — query + documents. Not a web search.
        client.rerank.create(model=rerank, query="q", documents=["a"])

    image_sku = pick(
        lambda model: "turbo" not in str(model["id"]).lower()
        and (
            surface_of(model).get("path") == "/v1/images/generations"
            or (
                "image_generation" in caps(model)
                and not str(model["id"]).startswith(("gemini/", "gemini-"))
            )
        )
    )
    if image_sku:
        # images.generate — 300s unless timeout= is set on Tonia(...)
        # Tenant dial (`1k`/`2k`/`4k`) or OpenAI size. Do not send resolution.
        # Image 2 maps 2k to 1536x1024. Image 2.5 2k is 2048x2048.
        client.images.generate(
            model=image_sku,
            prompt="Draw a red fox",
            n=1,
            size="2k",
            **({"quality": "auto"} if "gpt-image-2.5" in image_sku.lower() else {}),
        )

    gemini_image = pick(
        lambda model: surface_of(model).get("family") == "image"
        and surface_of(model).get("path") == "/v1/interactions"
        or (
            str(model["id"]).startswith("gemini/")
            and "image" in str(model["id"]).lower()
        )
    )
    if gemini_image:
        # Gemini image SKUs — /v1/interactions (not images.generate)
        client.interactions.create(
            model=gemini_image,
            input="Draw a red fox",
            stream=False,
        )

    tts = pick(
        lambda model: (
            surface_of(model).get("path") == "/v1/audio/speech"
            or "audio_speech" in caps(model)
        )
        and not str(model["id"]).startswith("gemini/")
        and "realtime" not in str(model["id"]).lower()
    )
    if tts:
        speech_voice = "" if str(tts).startswith("fish_") or "s2.1-pro" in str(tts) else "alloy"
        client.audio.speech.create(model=tts, input="Bonjour", voice=speech_voice)

    stt = pick(
        lambda model: (
            surface_of(model).get("path") == "/v1/audio/transcriptions"
            or "audio_transcription" in caps(model)
        )
        and not str(model["id"]).startswith("gemini/")
        and "realtime" not in str(model["id"]).lower()
    )
    if stt:
        client.audio.transcriptions.create(
            model=stt,
            file=open("clip.wav", "rb").read(),
            filename="clip.wav",
        )

    gemini_token_audio = pick(
        lambda model: str(model["id"]).startswith("gemini/")
        and surface_of(model).get("family") in ("speech", "transcription")
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
    except AgentBlockError:
        pass  # Open Policies → profile → Advanced — Agent controls.
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

    live = next(
        (
            model["id"]
            for model in rows
            if model["id"] in {"gpt-live-1", "openai/gpt-live-1"}
            or "realtime" in caps(model)
        ),
        None,
    )
    if live:
        with client.realtime.connect(model="gpt-live-1") as session:
            session.send_text("Reply with the single word ok.")
            session.wait_turn(timeout=60)
        if any(model["id"] in {"gemini-3.5-transcribe-live", "gemini/gemini-3.5-transcribe-live"} for model in rows):
            with client.realtime.connect(
                provider="gemini",
                model="gemini-3.5-transcribe-live",
                mode="native",
                transcripts=True,
            ) as captions:
                captions.send_audio_append(b"\x00\x00" * 1600, mime="audio/pcm;rate=16000")
                captions.send_audio_commit(mime="audio/pcm;rate=16000")
                captions.recv(timeout=60)

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
