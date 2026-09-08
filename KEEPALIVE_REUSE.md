# H2 keep-alive reuse timeout

The baseline is hyper-util v0.1.19
(`d5740116a55cbf7af13d1142b365c56b1d684f3a`), preserved on
`upstream/v0.1.19`. This branch uses a fixed revision of the companion hyper
v1.8.1 fork, which pins h2 to 0.4.13. The h2 fork requires no protocol changes.

```rust
use std::time::Duration;
use hyper_util::client::legacy::Client;
use hyper_util::rt::{TokioExecutor, TokioTimer};

let mut builder = Client::builder(TokioExecutor::new());
builder
    .timer(TokioTimer::new())
    .http2_keep_alive_interval(Some(Duration::from_secs(10)))
    .http2_keep_alive_reuse_timeout(Some(Duration::from_secs(5)))
    .http2_keep_alive_timeout(Duration::from_secs(60))
    .http2_keep_alive_while_idle(true);
// Continue with the application's existing connector and build() call.
```

The new option defaults to `None`. The soft timeout starts with the existing
PING timeout phase, not at network failure or request submission. Hyper invokes
the connection's observer once; it holds only that connection's PoisonPill.
The pool then filters the retired entry through its existing is_open checks.
No pool-wide state, retry rule, or h2 wire behavior is changed.

Existing requests and response bodies retain the original 60-second PING timeout.
A late ACK permits their completion but never reinstates the retired connection.
New requests establish replacement connections through normal pool coordination.
Already checked-out requests can race with retirement. This option does not
proactively connect or automatically replay requests. Network failures can still
cause replacement connections to fail.

When keep-alive is enabled, the soft timeout must be greater than zero and less
than the final hard timeout. Hyper validates at handshake, independent of setter
order. Keep-alive disabled means the soft setting has no effect.

## Validation

```sh
cargo tree -i hyper --features full
cargo tree -i h2 --features full
cargo test --features full --test keepalive_reuse
cargo test --features full --lib
cargo test --features full --test legacy_client
```

The dedicated tests use real H2 over duplex IO with both directions paused,
without EOF/reset, and a Tokio clock shared by TokioTimer. They cover early ACK,
default-disabled behavior, soft-deadline wakeup without IO, concurrent replacement
requests, late ACK with pending response headers and streaming body, adaptive
window operation, and the original 60-second hard deadline after retirement.

For an application already using registry dependencies, patch hyper-util to a
fixed commit of this fork in the application workspace. If it also directly
depends on hyper, patch hyper to the exact revision named in this Cargo.toml so
direct hyper types and hyper-util's hyper types share one package source.
