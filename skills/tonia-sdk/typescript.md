# TypeScript — `@tonia/sdk`

Need a key first? [Portal key setup](setup.md). Pointing Cursor at tonia?
[Coding tools](tools.md).

Read [`compatibility.json`](compatibility.json) before installing.

```bash
# status: local-staging — build the sibling tree, then depend on it
#   cd ../typescript-sdk && npm install && npm run build
npm add ../typescript-sdk
# or: npm add /path/to/tonia-router/typescript-sdk

# status: published (not yet) — then: npm add @tonia/sdk
```

Needs Node.js 22 or newer. Node 18 and 20 are end-of-life — do not
target them. Active LTS is 24. Build tooling uses TypeScript 6.0.

```ts
import {
  EntitlementError,
  PolicyBlockError,
  RateLimitError,
  Tonia,
} from "@tonia/sdk";

const client = new Tonia({
  apiKey: process.env.TONIA_API_KEY!,
  // baseURL defaults to https://pass.tonia.ca:8443
  defaultHeaders: {
    "HTTP-Referer": "https://example.com",
    "X-Tonia-Title": "Example App",
  },
});

await client.status.get();
await client.catalogue.list();
const publicModels = await client.publicModels.list();
await client.publicModelCategories.list();
if (publicModels.data[0]) {
  await client.publicModels.get(publicModels.data[0].id);
}

const { data } = await client.models.list();
const ids = data.map((model) => model.id);
if (!ids[0]) {
  throw new Error("empty allowlist — do not guess a model id");
}
const pick = (test: (id: string) => boolean) => ids.find(test);

await client.models.get(ids[0]);

await client.chat.completions.create({
  model: ids[0],
  messages: [{ role: "user", content: "Bonjour" }],
});

for await (const event of client.chat.completions.stream({
  model: ids[0],
  messages: [{ role: "user", content: "Bonjour" }],
})) {
  // event.json is a provider-shaped chunk when present
}

await client.responses.create({ model: ids[0], input: "Bonjour" });

const anthropic = pick((id) => id.startsWith("anthropic/"));
if (anthropic) {
  // sends x-api-key automatically
  await client.messages.create({
    model: anthropic,
    max_tokens: 256,
    messages: [{ role: "user", content: "Bonjour" }],
  });
}

const embedding = pick((id) => id.includes("embedding"));
if (embedding) {
  await client.embeddings.create({ model: embedding, input: "Bonjour" });
}

const rerank = pick((id) => id.includes("rerank"));
if (rerank) {
  await client.rerank.create({ model: rerank, query: "q", documents: ["a"] });
}

const imageSku = pick(
  (id) => /^(openai|xai|stepfun)\//.test(id) && /image/i.test(id),
);
if (imageSku) {
  // images.generate — 300s abort unless timeout is set on Tonia
  await client.images.generate({
    model: imageSku,
    prompt: "Draw a red fox",
    n: 1,
  });
}

const geminiImage = pick((id) => id.startsWith("gemini/") && /image/i.test(id));
if (geminiImage) {
  // Gemini image SKUs — /v1/interactions (not images.generate)
  await client.interactions.create({
    model: geminiImage,
    input: "Draw a red fox",
    stream: false,
  });
}

try {
  await client.chat.completions.create({
    model: ids[0],
    messages: [{ role: "user", content: "Bonjour" }],
  });
} catch (err) {
  if (err instanceof PolicyBlockError) {
    // Bind a redact-mode profile in the portal, then retry. Do not set a header.
  } else if (err instanceof RateLimitError && err.retryable) {
    // wait err.retryAfterSeconds, then retry once
  } else if (err instanceof EntitlementError) {
    // quota: retryable after Retry-After. budget (*_budget_exhausted): do not retry
  } else {
    throw err;
  }
}

client.lastLimits;
```

Use `.stream()` for SSE — do not pass `stream: true` to `create()` unless you
want the overload that returns an async generator. Prefer `.stream()`.

The SDK does not auto-retry. See [Errors and DLP](errors-and-dlp.md).
LLM `tools` are a passthrough body field — [Coding tools](tools.md).

Confirm signatures against the installed package and
[`tonia-api`](https://github.com/tonia-router/tonia-api).

SaaS apps store their own `messages[]`, meter provider `usage`, and read
`lastLimits` (only when usage is ≥ 80%). Billing and keys stay in the
[portal](https://portal.tonia.ca). Cookbook:
[`sdk-examples`](https://github.com/tonia-router/sdk-examples) `09-saas-integrator`.

Gemini image generate uses a string `input`. Edit uses
`[{ type: "text", text: "..." }, { type: "image", mime_type: "image/png", data: rawB64 }]`
(`data` is raw base64). Parse `model_output` image parts from the native
response. See the TypeScript SDK README **Images** section.
