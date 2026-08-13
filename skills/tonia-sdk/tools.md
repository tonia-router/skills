# Coding tools

Two different jobs. Do not mix them.

| Job | What to install | Credential |
| --- | --- | --- |
| Point Cursor / Claude Code / Codex at tonia | Nothing extra — base URL + `tonia_sk_…` | Same key from [setup](setup.md) |
| Call tonia from **your** app | `@tonia-router/sdk` or `tonia` | `TONIA_API_KEY` |

The official SDK is for application code. Cursor’s OpenAI override, Claude
Code, and Codex talk HTTP themselves — do not add `@tonia-router/sdk` to those
tools.

Production address: `https://pass.tonia.ca:8443`. Override only for an
explicit local or on-prem host. Port **8443** is required — `https://pass.tonia.ca`
on 443 is not Pass. Model ids must match `GET /v1/models` for this key
(or the [models page](https://tonia.ca/models) when configuring a tool
before the first SDK call).

## Cursor

Settings → Models:

1. In **Add or search model**, type the exact runtime id, then **Add**.
2. Paste the `tonia_sk_…` key.
3. Turn on **Override OpenAI Base URL** and set it to
   `https://pass.tonia.ca:8443/v1`.

Cursor is OpenAI-shaped (`Authorization: Bearer`). Repeat for each model id
the profile allows. Those ids are OpenAI-shaped (`anthropic/claude-…`).
Claude Code lists with `x-api-key` and sees Anthropic-shaped ids (`claude-…`,
no prefix). Do not paste one style into the other tool.

## Claude Code (CLI)

```bash
export ANTHROPIC_BASE_URL="https://pass.tonia.ca:8443"
export ANTHROPIC_AUTH_TOKEN="tonia_sk_…"
unset ANTHROPIC_API_KEY
```

Do **not** append `/v1` — the CLI adds it. Prefer `ANTHROPIC_AUTH_TOKEN` so
a leftover Anthropic Console key does not win. Restart the CLI, then
`/status`. This is the terminal CLI, not the Claude Desktop app (Desktop
only lists Claude-looking names).

## Codex CLI

```bash
export OPENAI_BASE_URL="https://pass.tonia.ca:8443/v1"
export OPENAI_API_KEY="tonia_sk_…"
```

Or the matching `[model_providers.…]` block in `~/.codex/config.toml`.
Include `/v1` once.

## Other tools

OpenAI-shaped clients (Cline, Continue `provider: openai`, LangChain,
n8n, …): base URL `https://pass.tonia.ca:8443/v1`, key `tonia_sk_…`.
Anthropic-shaped clients: base URL `https://pass.tonia.ca:8443` (no `/v1`),
`x-api-key` / `ANTHROPIC_API_KEY`.

Full recipes: [tonia.ca docs — Connect tools](https://tonia.ca/docs).

## LLM function tools (app code)

The SDK does not run a tool loop. Pass upstream-shaped `tools` on
`chat.completions.create`, `messages.create`, or `responses.create`.

```ts
await client.chat.completions.create({
  model: ids[0],
  messages: [{ role: "user", content: "Weather in Montréal?" }],
  tools: [
    {
      type: "function",
      function: {
        name: "get_weather",
        description: "Weather for a city",
        parameters: {
          type: "object",
          properties: { city: { type: "string" } },
          required: ["city"],
        },
      },
    },
  ],
});
```

If the model returns `tool_calls`, execute them in **your** process, append
the tool results, and call `create` again. See
[`sdk-examples`](https://github.com/tonia-router/sdk-examples) `08-tools-passthrough`.
