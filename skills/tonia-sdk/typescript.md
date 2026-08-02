# TypeScript — `@tonia/sdk`

```bash
npm add @tonia/sdk
# or: pnpm add @tonia/sdk / yarn add @tonia/sdk
```

```ts
import { Tonia, PolicyBlockError } from "@tonia/sdk";

const client = new Tonia({
  apiKey: process.env.TONIA_API_KEY!,
  // baseURL defaults to https://pass.tonia.ca
  defaultHeaders: {
    "HTTP-Referer": "https://example.com",
    "X-Tonia-Title": "Example App",
  },
});

await client.status.get();
await client.publicModels.list();
await client.models.list();

await client.chat.completions.create({
  model: "openai/gpt-4.1-mini",
  messages: [{ role: "user", content: "Bonjour" }],
});

for await (const event of client.chat.completions.stream({
  model: "openai/gpt-4.1-mini",
  messages: [{ role: "user", content: "Bonjour" }],
})) {
  // event.json is a provider-shaped chunk when present
}

// Anthropic-shaped helper sends x-api-key automatically
await client.messages.create({
  model: "anthropic/claude-sonnet-4-5",
  max_tokens: 256,
  messages: [{ role: "user", content: "Bonjour" }],
});

// Raw helper — supported Pass path prefixes only
await client.request("GET", "/v1/public/catalogue");

// Soft-limit headers from the last successful call
client.lastLimits;
```

Confirm signatures against the installed package and
[`tonia-api`](https://github.com/tonia-router/tonia-api).
