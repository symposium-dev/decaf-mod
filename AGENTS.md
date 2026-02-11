# Decaf Mod

Debouncing proxy for ACP. Agents often send `AgentMessageChunk` notifications word-by-word; Decaf coalesces these into fewer, larger chunks sent at a configurable interval.

## Project structure

- `src/lib.rs` — The entire library implementation. Exports `Decaf` struct.
- `src/main.rs` — Binary entry point. Reads optional interval-ms CLI arg (default 100), connects to stdio via `ByteStreams`.
- `tests/debounce.rs` — Integration test with a `FastWordAgent` that sends 20 words as individual chunks, verifies coalescing.

## How it works

`Decaf::new(Duration)` creates the proxy. `Decaf::run(transport)` starts it using the SACP `Proxy` builder.

Three flush triggers:
1. **Timer tick** — A `with_spawned` background task calls `flush_all` at the configured interval.
2. **Non-text notification** — When a non-`AgentMessageChunk` notification arrives from the agent, the buffer is flushed first to preserve ordering, then the notification is forwarded.
3. **PromptResponse** — When the agent responds to a `PromptRequest`, all buffers are flushed before the response reaches the client (so no text is lost).

Per-session state is held in `Arc<Mutex<HashMap<SessionId, BufferedSession>>>`. The notification handler buffers into shared state; the spawned timer task reads from it. The mutex synchronizes handler vs spawned task (the handler is called sequentially by the event loop, so no self-races).

`Decaf` implements `ConnectTo<Conductor>` so it plugs into proxy chains: `ProxiesAndAgent::new(agent).proxy(Decaf::new(...))`.

## Binary usage

The binary speaks SACP JSON-RPC over stdin/stdout via `ByteStreams` with `tokio_util::compat`. It takes one optional positional argument: the debounce interval in milliseconds (default 100).

```
decaf-mod [interval_ms]
```

## Library usage

```rust
use decaf_mod::Decaf;
use std::time::Duration;

// In a proxy chain
ProxiesAndAgent::new(my_agent)
    .proxy(Decaf::new(Duration::from_millis(100)));

// Standalone
Decaf::new(Duration::from_millis(100))
    .run(transport)
    .await?;
```

## CI and releases

Releases are automated via the `symposium-dev/package-agent-extension` workflow in `.github/workflows/release.yml`. On a GitHub release event, it cross-compiles for macOS (ARM/x64), Linux (ARM/x64, musl-static), and Windows (x64), then uploads platform archives and an `extension.json` manifest.

The `[package.metadata.symposium]` section in `Cargo.toml` configures how the conductor spawns the binary. Currently `args = ["100"]` passes the default debounce interval.

Versioning uses [release-plz](https://release-plz.ieni.dev/) with conventional commits:
- `fix: ...` — patch bump
- `feat: ...` — minor bump
- `feat!: ...` or `BREAKING CHANGE:` — major bump

## Build and test

```
cargo test
```

This runs the integration test (`test_decaf_coalesces_chunks`), unit tests, and doc-tests.
