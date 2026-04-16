---
# Rust Project Setup KB

## Overview
Practical setup and configuration notes for the Nverse Rust project, including toolchain details, dependencies, module layout, and common pitfalls discovered during development.

## Key Information

- **Rust version:** 1.94.0, installed via rustup
- **Cargo env must be sourced before use in a new shell session:**
  ```bash
  source "$HOME/.cargo/env"
  ```
  Without this, `cargo` and `rustc` will not be on `PATH`.

- **Project module structure:** Each module lives under `src/<module>/mod.rs` and must be declared in `src/lib.rs`:
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

## Cargo.toml Dependencies

```toml
[dependencies]
serde       = { version = "1", features = ["derive"] }
toml        = "0.8"
macroquad   = "0.4"
pcap        = "2"
sysinfo     = "0.30"
log         = "0.4"
env_logger  = "0.11"
thiserror   = "1"
parking_lot = "0.12"
```

## Commands / Examples

```bash
# Source Cargo environment (required once per shell session)
source "$HOME/.cargo/env"

# Build (debug)
cargo build

# Build (release)
cargo build --release

# Run lib tests only (avoids binary linker issues with macroquad/pcap)
cargo test --lib

# Lint
cargo clippy

# Format
cargo fmt
```

## Notes on `cargo test --lib`

Running plain `cargo test` will attempt to compile and link the binary target, which pulls in macroquad and pcap. Both require native system libraries (OpenGL, X11, libpcap). On headless CI or restricted environments this fails. Using `--lib` restricts compilation to the library crate and its unit tests, avoiding those link requirements.

## Release Build Characteristics

- **Release binary size:** ~3.7 MB — macroquad compiles to a small, self-contained static binary.
- **Release build time:** approximately 1 minute on a typical development machine.
- **Release binary location:** `target/release/nverse`

## libpcap Workaround (when `libpcap-dev` is not installed)

If `libpcap-dev` (the development headers/stub) is unavailable but the runtime library is present, create a symlink and point Cargo at it:

```bash
mkdir -p ~/.local/lib
ln -sf /usr/lib/x86_64-linux-gnu/libpcap.so.1.10.4 ~/.local/lib/libpcap.so
```

Then add to `.cargo/config.toml` in the project root:

```toml
[build]
rustflags = ["-L", "/home/flintlock/.local/lib"]
```

This lets the `pcap` crate find `libpcap.so` at link time without a system-wide dev package.

## macroquad UI Panel Pitfalls

- **Gate panel rendering on data availability.** UI panels rendered unconditionally (even when empty) will cover the 3D scene. Check before rendering:
  ```rust
  if selected_key.is_some() {
      panel.render(...);
  }
  ```
- **Panel height fraction of 0.45 (45% of window) is too large** for info overlays. A fraction of 0.20–0.25 works well and leaves the 3D scene visible.

## macroquad Entry Point and Test Constraints

- macroquad requires a specific async entry point:
  ```rust
  #[macroquad::main("Nverse")]
  async fn main() {
      // game loop here
  }
  ```
- All macroquad draw calls (`draw_sphere`, `draw_line_3d`, etc.) **must** be issued inside the macroquad async loop. Calling them outside that context panics or produces undefined behaviour.
- Tests that invoke macroquad draw calls must be excluded from the normal test run. Use either:
  - `#[ignore]` on the test function, or
  - `#[cfg(not(test))]` to gate the draw-call code entirely.

## State Module Constructor Signatures

These were discovered to differ from what the IOC specs implied:

- **`TimeoutEngine::new()`** takes three arguments:
  ```rust
  TimeoutEngine::new(
      arc_connection_map: Arc<ConnectionMap>,
      arc_publisher: Arc<ConnectionEventPublisher>,
      config: &ConnectionConfig,
  )
  ```
- **`ConnectionMap::new()`** takes `max_connections: u32` directly, **not** a `ConnectionConfig`:
  ```rust
  ConnectionMap::new(max_connections: u32)
  ```
