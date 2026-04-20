---
# Rust Project Setup KB

## Overview
Practical setup and configuration notes for Rust projects, including toolchain management, common dependencies, module layout patterns, and common pitfalls.

## Key Information

- **Rust version:** Installed via rustup. Check your version with:
  ```bash
  rustc --version
  ```

- **Cargo env must be sourced before use in a new shell session:**
  ```bash
  source "$HOME/.cargo/env"
  ```
  Without this, `cargo` and `rustc` will not be on `PATH`.

- **Project module structure:** Each module lives under `src/<module>/mod.rs` (or `src/<module>.rs` for simpler projects) and must be declared in `src/lib.rs`:
  ```rust
  pub mod config;
  pub mod capture;
  pub mod protocol;
  // etc.
  ```

- **`AtomicF32` does not exist in `std`** — pack/unpack floats via `AtomicU32` with bit-casting:
  ```rust
  use std::sync::atomic::{AtomicU32, Ordering};

  // Store
  atomic.store(value.to_bits(), Ordering::Relaxed);

  // Load
  let value = f32::from_bits(atomic.load(Ordering::Relaxed));
  ```

- **Prefer `parking_lot::{RwLock, Mutex}`** over `std::sync` equivalents — better performance, no poisoning, simpler API.

- **Raw string literals:** If the string body contains the sequence `"#`, the `r#"..."#` delimiter is ambiguous. Use `r##"..."##` (or more `#`s) to avoid parse errors:
  ```rust
  // This will fail if content contains "#:
  let s = r#"some "# text"#;

  // Use extra delimiters instead:
  let s = r##"some "# text"##;
  ```

- **Common dependencies for data/config/CLI projects:**
  ```toml
  [dependencies]
  serde       = { version = "1", features = ["derive"] }
  toml        = "0.8"
  thiserror   = "1"
  parking_lot = "0.12"
  log        = "0.4"
  env_logger = "0.11"
  ```

- **Common dependencies for GUI/graphics projects:**
  ```toml
  [dependencies]
  macroquad = "0.4"
  # plus system libraries: libGL, libEGL, libX11, libxkbcommon
  ```

- **Common dependencies for network/system projects:**
  ```toml
  [dependencies]
  pcap     = "2"    # requires libpcap-dev
  sysinfo  = "0.30"
  # plus elevated privileges (root) or CAP_NET_RAW
  ```

## Commands

```bash
# Source Cargo environment (required once per shell session)
source "$HOME/.cargo/env"

# Build (debug)
cargo build

# Build (release)
cargo build --release

# Run lib tests only (avoids binary linker issues)
cargo test --lib

# Lint
cargo clippy

# Format
cargo fmt
```

## Notes on `cargo test --lib`

Running plain `cargo test` will attempt to compile and link the binary target. Some crates (GUI libraries, native bindings) require system libraries that may not be available in headless/CI environments. Using `--lib` restricts compilation to the library crate and its unit tests, avoiding those link requirements.

## Native Library Workarounds

When a development package is unavailable but the runtime library exists, you can often work around it:

```bash
# Example: libpcap runtime exists but headers missing
mkdir -p ~/.local/lib
ln -sf /usr/lib/x86_64-linux-gnu/libpcap.so.1.10.4 ~/.local/lib/libpcap.so
```

Then add to `.cargo/config.toml`:

```toml
[build]
rustflags = ["-L", "/home/<user>/.local/lib"]
```

## Common Pitfalls

- **Gate UI rendering on data availability.** UI elements rendered unconditionally (even when empty) may cover the main view. Check before rendering:
  ```rust
  if selected_key.is_some() {
      panel.render(...);
  }
  ```

- **Library entry points.** Some libraries (like macroquad) require a specific async entry point:
  ```rust
  #[macroquad::main("MyApp")]
  async fn main() {
      // game/app loop here
  }
  ```

- **Draw calls must be in context.** Some graphics libraries require all draw calls to be issued inside their specific async loop context. Calling them outside that context panics or produces undefined behavior.

- **Tests with native dependencies.** Tests that invoke library draw calls or native functions must be excluded from normal test runs. Use either:
  - `#[ignore]` on the test function, or
  - `#[cfg(not(test))]` to gate the library code entirely.

- **Constructor signatures may differ from specs.** When implementing, verify constructor signatures in the actual source code — documentation or specs may be outdated.