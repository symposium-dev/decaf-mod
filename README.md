# Decaf

Debouncing proxy for ACP. Agents often send `AgentMessageChunk` notifications word-by-word, creating a flood of tiny messages. Decaf coalesces these chunks, forwarding a single combined chunk every N milliseconds instead.

## As a library

```rust
use decaf::Decaf;
use std::time::Duration;

// In a proxy chain
ProxiesAndAgent::new(my_agent)
    .proxy(Decaf::new(Duration::from_millis(100)));

// Standalone
Decaf::new(Duration::from_millis(100))
    .run(transport)
    .await?;
```

`Decaf` implements `ConnectTo<Conductor>`, so it plugs directly into SACP proxy chains.

## As a binary

```
decaf [interval_ms]
```

Runs as an ACP proxy over stdin/stdout. The optional argument sets the debounce interval in milliseconds (default: 100).

## How it works

Decaf intercepts `AgentMessageChunk` notifications from the agent side and buffers the text content per session. Buffered text is flushed to the client on three triggers:

- **Timer tick** at the configured interval
- **Non-text notification** from the agent (flush first to preserve ordering, then forward)
- **PromptResponse** from the agent (flush before forwarding so no text is lost)

## License

Licensed under either of [Apache License, Version 2.0](LICENSE-APACHE) or [MIT License](LICENSE-MIT) at your option.
