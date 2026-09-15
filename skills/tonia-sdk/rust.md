# Rust — `tonia-sdk`

The crate is **`tonia-sdk` 0.4.0** (`tonia-sdk-rs/0.4.0`) on
`github.com/tonia-router/rust-sdk` `main` (tag `v0.4.0`). It is **not** on
crates.io. See [`compatibility.json`](compatibility.json)
(`status: unpublished`). Do not `cargo add tonia-sdk` from the registry.

```toml
tonia-sdk = { git = "https://github.com/tonia-router/rust-sdk", tag = "v0.4.0" }
# local Tonia tree:
# tonia-sdk = { path = "../tonia-router/rust-sdk" }
```

Requires Rust **1.85+**. Env is `TONIA_API_KEY` and `TONIA_REALTIME_URL`
only. Pass `base_url` for hosted DEV — the crate does not read
`TONIA_BASE_URL`. Timeouts follow Python: 60s default; 300s on images /
speech / STT / `interactions.create` and `interactions.stream`; a ctor
`timeout` overrides the 300s.

Parity work is [plan 35](../../../../PLANS/04-MVP-execution-plans/launch-prod-tenant-path/35-rust-sdk-parity-execution-plan.md).
Unary helpers and `request()` exist. `.stream()` still buffers (do not use
it for cookbook 02). `realtime.connect`, multipart STT, and raw speech
bytes are not finished. For a complete published surface use
[TypeScript](typescript.md) or [Python](python.md). The same
[portal key](setup.md) and [coding tools](tools.md) steps apply.
