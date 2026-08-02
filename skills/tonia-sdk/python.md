# Python — `tonia`

```bash
pip install tonia
```

```python
import os
from tonia import AsyncTonia, Tonia, PolicyBlockError

with Tonia(api_key=os.environ["TONIA_API_KEY"]) as client:
    client.status.get()
    client.public_models.list()
    client.models.list()

    client.chat.completions.create(
        model="openai/gpt-4.1-mini",
        messages=[{"role": "user", "content": "Bonjour"}],
    )

    for event in client.chat.completions.stream(
        model="openai/gpt-4.1-mini",
        messages=[{"role": "user", "content": "Bonjour"}],
    ):
        pass  # event.json is a provider-shaped chunk when present

    # Anthropic-shaped helper sends x-api-key automatically
    client.messages.create(
        model="anthropic/claude-sonnet-4-5",
        max_tokens=256,
        messages=[{"role": "user", "content": "Bonjour"}],
    )

    # Raw helper — supported Pass path prefixes only
    client.request("GET", "/v1/public/catalogue")

    client.last_limits  # soft-limit headers from the last successful call

async with AsyncTonia(api_key=os.environ["TONIA_API_KEY"]) as client:
    await client.models.list()
```

Confirm signatures against the installed package and
[`tonia-api`](https://github.com/tonia-router/tonia-api).
