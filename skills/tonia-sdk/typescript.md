# TypeScript — `@tonia-router/sdk`

Need a key first? [Portal key setup](setup.md). Pointing Cursor at tonia?
[Coding tools](tools.md).

Read [`compatibility.json`](compatibility.json) before installing.

```bash
# packages are not on npm yet — build the sibling tree, then depend on it
#   cd ../typescript-sdk && npm install && npm run build
npm add ../typescript-sdk
# or: npm add /path/to/tonia-router/typescript-sdk

# once published: npm add @tonia-router/sdk
```

Needs Node.js 22 or newer. Node 18 and 20 are end-of-life — do not
target them. Active LTS is 24. Build tooling uses TypeScript 6.0.

```ts
import { readFile } from "node:fs/promises";
import {
  AgentBlockError,
  EntitlementError,
  PolicyBlockError,
  RateLimitError,
  Tonia,
} from "@tonia-router/sdk";

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
  throw new Error("this key has no models; check the profile allowlist in the portal");
}
// models.list() is Bearer / OpenAI-shaped (anthropic/claude-…).
// messages.create sends x-api-key but still takes that same id.
const caps = (model: (typeof data)[number]) => model.capabilities ?? [];
const surfaceOf = (model: (typeof data)[number]) => model.surface ?? {};
const pick = (test: (model: (typeof data)[number]) => boolean) =>
  data.find(test)?.id;

await client.models.get(ids[0]);

await client.chat.completions.create({
  model: ids[0],
  messages: [{ role: "user", content: "Bonjour" }],
});

const ac = new AbortController();
for await (const event of client.chat.completions.stream(
  {
    model: ids[0],
    messages: [{ role: "user", content: "Bonjour" }],
  },
  { signal: ac.signal },
)) {
  // event.json is a provider-shaped chunk when present
}
// Hang up: ac.abort(), or break the loop.

await client.responses.create({ model: ids[0], input: "Bonjour" });

const anthropic = pick((model) => model.id.startsWith("anthropic/"));
if (anthropic) {
  // sends x-api-key automatically
  await client.messages.create({
    model: anthropic,
    max_tokens: 256,
    messages: [{ role: "user", content: "Bonjour" }],
  });
}

const embedding = pick((model) => {
  if (surfaceOf(model).family === "embeddings") return true;
  if (caps(model).includes("embeddings")) return true;
  const id = model.id.toLowerCase();
  return id.includes("embed") && !id.includes("rerank");
});
if (embedding) {
  // Tenant /v1/embeddings — send model + input. Not /v2/embed.
  await client.embeddings.create({ model: embedding, input: "Bonjour" });
}

const rerank = pick(
  (model) =>
    surfaceOf(model).family === "rerank" ||
    caps(model).includes("rerank") ||
    model.id.toLowerCase().includes("rerank"),
);
if (rerank) {
  // Tenant /v1/rerank — query + documents. Not a web search.
  await client.rerank.create({ model: rerank, query: "q", documents: ["a"] });
}

const systemone = pick(
  (model) =>
    surfaceOf(model).family === "systemone" ||
    caps(model).includes("systemone") ||
    model.id.endsWith("jev-latest"),
);
if (systemone) {
  // Tenant /v1/systemone — state + questions. No typed helper. No stream.
  await client.request("POST", "/v1/systemone", {
    model: systemone,
    state: "The sky is blue.",
    questions: { color: { type: "noul", instructions: "Is the sky blue?" } },
  });
}

const imageSku = pick((model) => {
  if (/turbo/i.test(model.id)) return false;
  return (
    surfaceOf(model).path === "/v1/images/generations" ||
    (caps(model).includes("image_generation") &&
      !model.id.startsWith("gemini/") &&
      !model.id.startsWith("gemini-"))
  );
});
if (imageSku) {
  // images.generate — 300s abort unless timeout is set; { signal } hangs up early
  // Tenant dial (`1k`/`2k`/`4k`) or OpenAI size. Do not send resolution.
  // Image 2 maps 2k to 1536x1024. Image 2.5 2k is 2048x2048.
  await client.images.generate({
    model: imageSku,
    prompt: "Draw a red fox",
    n: 1,
    size: "2k",
    ...( /gpt-image-2\.5/i.test(imageSku) ? { quality: "auto" as const } : {}),
  });
}

const geminiImage = pick(
  (model) =>
    (surfaceOf(model).family === "image" &&
      surfaceOf(model).path === "/v1/interactions") ||
    (model.id.startsWith("gemini/") && /image/i.test(model.id)),
);
if (geminiImage) {
  // Gemini image SKUs — /v1/interactions (not images.generate)
  await client.interactions.create({
    model: geminiImage,
    input: "Draw a red fox",
    stream: false,
  });
}

const tts = pick(
  (model) =>
    (surfaceOf(model).path === "/v1/audio/speech" ||
      caps(model).includes("audio_speech")) &&
    !model.id.startsWith("gemini/") &&
    !/realtime/i.test(model.id),
);
if (tts) {
  const voice = tts.startsWith("fish_") || tts.includes("s2.1-pro") ? "" : "alloy";
  await client.audio.speech.create({ model: tts, input: "Bonjour", voice });
}

const stt = pick(
  (model) =>
    (surfaceOf(model).path === "/v1/audio/transcriptions" ||
      caps(model).includes("audio_transcription")) &&
    !model.id.startsWith("gemini/") &&
    !/realtime/i.test(model.id),
);
if (stt) {
  await client.audio.transcriptions.create({
    model: stt,
    file: await readFile("clip.wav"),
    filename: "clip.wav",
  });
}

const geminiTokenAudio = pick(
  (model) =>
    model.id.startsWith("gemini/") &&
    (surfaceOf(model).family === "speech" ||
      surfaceOf(model).family === "transcription"),
);
if (geminiTokenAudio) {
  // Gemini token TTS/STT — never audio.speech / audio.transcriptions
  await client.interactions.create({
    model: geminiTokenAudio,
    input: "Bonjour",
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
  } else if (err instanceof AgentBlockError) {
    // Open Policies → profile → Advanced — Agent controls.
  } else if (err instanceof RateLimitError && err.retryable) {
    // wait err.retryAfterSeconds, then retry once
  } else if (err instanceof EntitlementError) {
    // quota: retryable after Retry-After. budget (*_budget_exhausted): do not retry
  } else {
    throw err;
  }
}

const live = pick(
  (model) =>
    model.id === "gpt-live-1" ||
    model.id === "openai/gpt-live-1" ||
    caps(model).includes("realtime"),
);
if (live) {
  const session = await client.realtime.connect({ model: "gpt-live-1" });
  session.sendText("Reply with the single word ok.");
  await session.waitTurn(60_000);
  session.close();
}
const captionsId = pick(
  (model) =>
    model.id === "gemini-3.5-transcribe-live" ||
    model.id === "gemini/gemini-3.5-transcribe-live",
);
if (captionsId) {
  const captions = await client.realtime.connect({
    provider: "gemini",
    model: "gemini-3.5-transcribe-live",
    mode: "native",
    transcripts: true,
  });
  captions.sendAudioAppend(new Uint8Array(3200), "audio/pcm;rate=16000");
  captions.sendAudioCommit(undefined, "audio/pcm;rate=16000");
  await captions.recv(60_000);
  captions.close();
}

client.lastLimits;
```

Use `.stream()` for SSE — do not pass `stream: true` to `create()` unless you
want the overload that returns an async generator. Prefer `.stream()`.
Pass `{ signal }` to hang up; `break` also cancels the body.

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
