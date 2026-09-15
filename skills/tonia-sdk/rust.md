# Rust — `tonia-sdk`

Need a key first? [Portal key setup](setup.md). Pointing Cursor at tonia?
[Coding tools](tools.md).

Read [`compatibility.json`](compatibility.json) before installing.

The crate is **`tonia-sdk` 0.4.1** (`tonia-sdk-rs/0.4.1`) on
`github.com/tonia-router/rust-sdk` `main` (tag `v0.4.1`). It is **not** on
crates.io. Do not `cargo add tonia-sdk` from the registry.

```toml
tonia-sdk = { git = "https://github.com/tonia-router/rust-sdk", tag = "v0.4.1" }
# local Tonia tree:
# tonia-sdk = { path = "../tonia-router/rust-sdk" }
```

Requires Rust **1.85+**. Env is `TONIA_API_KEY` and `TONIA_REALTIME_URL`
only. Pass `base_url` for hosted DEV — the crate does not read
`TONIA_BASE_URL`. Timeouts follow Python: 60s default; 300s on images /
speech / STT / `interactions.create` and `interactions.stream`; a ctor
`timeout` overrides the 300s.

```rust
use std::time::Duration;

use futures_util::StreamExt;
use serde_json::json;
use tonia_sdk::{
    ErrorKind, RealtimeConnect, TranscriptionFile, Tonia, DEFAULT_BASE_URL, IMAGE_TIMEOUT,
};

#[tokio::main]
async fn main() -> Result<(), tonia_sdk::ToniaError> {
    let client = Tonia::builder()
        .api_key(std::env::var("TONIA_API_KEY").expect("TONIA_API_KEY"))
        .base_url(DEFAULT_BASE_URL) // hosted DEV must pass this; env is not read
        .header("X-Tonia-Title", "example")
        .build();

    client.status.get().await?;
    client.catalogue.list().await?;
    let public_models = client.public_models.list().await?;
    client.public_model_categories.list().await?;
    if let Some(id) = public_models["data"][0]["id"].as_str() {
        client.public_models.get(id).await?;
    }

    let listed = client.models.list().await?;
    let rows = listed["data"].as_array().cloned().unwrap_or_default();
    let ids: Vec<_> = rows
        .iter()
        .filter_map(|model| model["id"].as_str().map(str::to_string))
        .collect();
    if ids.is_empty() {
        return Ok(());
    }

    // models.list() is Bearer / OpenAI-shaped (anthropic/claude-…).
    // messages.create sends x-api-key but still takes that same id.
    client.models.get(&ids[0]).await?;

    client
        .chat
        .completions
        .create(json!({
            "model": ids[0],
            "messages": [{ "role": "user", "content": "Bonjour" }],
        }))
        .await?;

    let mut stream = client.chat.completions.stream(json!({
        "model": ids[0],
        "messages": [{ "role": "user", "content": "Bonjour" }],
    }));
    while let Some(event) = stream.next().await {
        let _event = event?; // event.json is a provider-shaped chunk when present
    }

    client
        .responses
        .create(json!({ "model": ids[0], "input": "Bonjour" }))
        .await?;

    if let Some(anthropic) = ids.iter().find(|id| id.starts_with("anthropic/")) {
        client
            .messages
            .create(json!({
                "model": anthropic,
                "max_tokens": 256,
                "messages": [{ "role": "user", "content": "Bonjour" }],
            }))
            .await?;
    }

    client
        .embeddings
        .create(json!({ "model": "embed", "input": "Bonjour" }))
        .await
        .ok();
    client
        .rerank
        .create(json!({ "model": "rerank", "query": "q", "documents": ["a"] }))
        .await
        .ok();

    // images.generate / edit — JSON, 300s unless ToniaBuilder::timeout is set.
    client
        .images
        .generate(json!({
            "model": "gpt-image-2",
            "prompt": "Draw a red fox",
            "n": 1,
            "size": "2k",
        }))
        .await
        .ok();
    client
        .images
        .edit(json!({ "model": "gpt-image-2", "prompt": "Make it sit" }))
        .await
        .ok();

    // Gemini image SKUs — /v1/interactions (not images.generate)
    client
        .interactions
        .create(json!({
            "model": "gemini/gemini-2.5-flash-image",
            "input": "Draw a red fox",
            "stream": false
        }))
        .await
        .ok();

    let _speech = client
        .audio
        .speech
        .create(json!({
            "model": "gpt-4o-mini-tts",
            "input": "Bonjour",
            "voice": "alloy"
        }))
        .await?;
    client
        .audio
        .transcriptions
        .create(
            "gpt-transcribe",
            TranscriptionFile::try_from_user("clip.wav").expect("path, not a data URI"),
            Some("clip.wav"),
            &[],
        )
        .await
        .ok();

    match client
        .chat
        .completions
        .create(json!({
            "model": ids[0],
            "messages": [{ "role": "user", "content": "Bonjour" }],
        }))
        .await
    {
        Err(err) if err.kind == ErrorKind::PolicyBlock => {}
        Err(err) if err.kind == ErrorKind::AgentBlock => {}
        Err(err) if err.kind == ErrorKind::RateLimit && err.retryable == Some(true) => {
            let _wait = err.retry_after_seconds().unwrap_or(1);
        }
        Err(err) if err.kind == ErrorKind::Entitlement && err.retryable != Some(true) => {
            return Err(err);
        }
        other => {
            other?;
        }
    }

    let _ = client.last_limits();
    let _ = IMAGE_TIMEOUT; // Duration::from_secs(300)

    if ids.iter().any(|id| id == "gpt-live-1" || id == "openai/gpt-live-1") {
        let mut session = client
            .realtime
            .connect(RealtimeConnect {
                model: Some("gpt-live-1".into()),
                ..RealtimeConnect::default()
            })
            .await?;
        session.send_text("Reply with the single word ok.").await?;
        session.wait_turn(Duration::from_secs(60)).await?;
        session.close().await?;
    }

    // Escape hatch: JSON only. HTTP /v1/realtime is PathNotAllowedError.
    client
        .request(
            "POST",
            "/v1/audio/translations",
            Some(json!({ "model": "whisper" })),
            tonia_sdk::RequestOptions::default(),
        )
        .await
        .ok();
    Ok(())
}
```

`create(..., "stream": true)` is not a dual return type — call `.stream()`.
The same `.stream()` helper exists on `messages`, `responses`, and
`interactions` (300s). Public `Tonia::stream` is the TS-shaped SSE escape
hatch (`request()` stays JSON-only). Live also has `send_audio_append` /
`send_audio_commit` / `send_delegation_result` / `recv` (cookbook 15–16).
`request()` cannot multipart and cannot hit `/v1/realtime`. STT `data:` URIs
raise `TranscriptionError::InvalidFile` (`InvalidTranscriptionFile`, not a
`ToniaError` kind) before the network. Speech returns raw `Bytes` when
`Content-Type` is `audio/*` or `application/octet-stream`.

The SDK does not auto-retry. See [Errors and DLP](errors-and-dlp.md).
LLM `tools` are a passthrough body field — [Coding tools](tools.md).

Confirm signatures against the crate and
[`tonia-api`](https://github.com/tonia-router/tonia-api). Cookbook:
[`sdk-examples/rust`](https://github.com/tonia-router/sdk-examples) `01`–`16`.
