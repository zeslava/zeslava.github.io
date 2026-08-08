+++
title = "esp rust: setting up defmt"
date = 2026-07-06
description = "A note on setting up defmt"
+++

## Setting up defmt on ESP32-C6 (esp-hal + probe-rs)

Notes on migrating from `rtt-target` to `defmt` for logging over RTT.

## 1. Dependencies

```toml
[dependencies]
defmt     = "1.1.1"
defmt-rtt = "1.3.0"
```

`rtt-target` can be removed — `defmt-rtt` replaces it.

## 2. Linking: the `.defmt` section

Error at startup:

```
Error: Failed to parse defmt data
defmt version found, but no `.defmt` section - check your linker configuration
```

Cause: the `defmt.x` linker script (shipped with the `defmt-rtt` crate) isn't linked in.

Fix — add a flag to `.cargo/config.toml`:

```toml
[build]
rustflags = [
  "-C", "link-arg=-Tdefmt.x",
]
```

## 3. Transport initialization

`defmt` by itself only formats messages — where they're sent is decided by a separate transport crate.
Since the project already had an RTT channel (via `rtt-target`), the logical choice is `defmt-rtt`:

```rust
use defmt_rtt as _; // wires up RTT as the transport for defmt
```

Important: `rtt_target::rtt_init_print!()` must be removed — two ways of initializing the RTT channel conflict.

## 4. The mandatory timestamp

Without it the build fails at link time:

```
undefined symbol: _defmt_timestamp
```

Every defmt frame includes a timestamp per the protocol, so it needs to be set explicitly:

```rust
defmt::timestamp!(
    "{=u64:us}",
    esp_hal::time::Instant::now()
        .duration_since_epoch()
        .as_micros()
);
```

Here `:us` tells the host (probe-rs) to format the number as seconds with a fractional part
(e.g. `12.345678`), instead of printing raw microseconds.

If a timestamp isn't needed at all, an empty format can be left — the symbol still gets
defined for linking:

```rust
defmt::timestamp!("");
```

## 5. "Builds, but prints nothing"

The likely cause is the log level not being set via the `DEFMT_LOG` environment variable (the `defmt` analog of `RUST_LOG` from `env_logger`).
By default, `info` level doesn't pass through.

Quick check:

```bash
DEFMT_LOG=info cargo run --release
```

To avoid setting the variable every time, put it in `.cargo/config.toml`:

```toml
[env]
DEFMT_LOG = "info"
```

## Final checklist

- [x] `defmt` + `defmt-rtt` in dependencies, `rtt-target` removed
- [x] `-Tdefmt.x` in `rustflags`
- [x] `use defmt_rtt as _;` in `main.rs`, old `rtt_init_print!()` removed
- [x] `defmt::timestamp!` defined (otherwise linking fails)
- [x] `DEFMT_LOG` set (directly or via `[env]` in `.cargo/config.toml`)
