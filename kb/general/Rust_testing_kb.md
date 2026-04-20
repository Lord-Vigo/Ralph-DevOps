---
# Rust Testing KB

## Overview
Conventions and patterns for writing and running unit tests in Rust projects.

## Key Information

- **Unit tests live inside the source file they test**, in a `#[cfg(test)]` module at the bottom:
  ```rust
  #[cfg(test)]
  mod tests {
      use super::*;

      #[test]
      fn test_something() {
          assert_eq!(1 + 1, 2);
      }
  }
  ```

- **`#[test]`** marks a function as a test case. The function must take no arguments and return `()` (or `Result<(), E>` for the `?` operator).

- **Tests that require elevated privileges** (e.g., raw packet capture) must be marked `#[ignore]` so they are skipped in normal runs:
  ```rust
  #[test]
  #[ignore] // requires root / CAP_NET_RAW
  fn test_live_capture() {
      // ...
  }
  ```
  Run them explicitly with: `cargo test -- --ignored`

- **`tempfile` crate** — use for tests that need temporary files or directories:
  ```toml
  [dev-dependencies]
  tempfile = "3"
  ```
  ```rust
  use tempfile::NamedTempFile;
  let mut f = NamedTempFile::new().unwrap();
  writeln!(f, "data").unwrap();
  ```

- **Assertion macros:**
  - `assert_eq!(left, right)` — equality, prints both values on failure
  - `assert_ne!(left, right)` — inequality
  - `assert!(condition)` — boolean condition
  - `assert_matches!(value, pattern)` — pattern match (requires nightly or import from `std::assert_matches`)

## Commands

```bash
# Run all lib unit tests (recommended — avoids binary linker issues)
cargo test --lib

# Run a specific test by name (substring match)
cargo test --lib test_something

# Show println! / eprintln! output during tests
cargo test --lib -- --nocapture

# Run a single named test with output visible
cargo test --lib test_something -- --nocapture

# Run tests that are normally ignored
cargo test --lib -- --ignored

# Run all tests including ignored ones
cargo test --lib -- --include-ignored
```

## Example Test Structure

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use tempfile::NamedTempFile;
    use std::io::Write;

    #[test]
    fn parses_valid_config() {
        let mut f = NamedTempFile::new().unwrap();
        writeln!(f, r##"inactive_timeout = 60"##).unwrap();
        let cfg = Config::load(f.path()).unwrap();
        assert_eq!(cfg.inactive_timeout, 60);
    }

    #[test]
    #[ignore] // requires CAP_NET_RAW
    fn live_capture_opens() {
        let cap = Capture::open("eth0").unwrap();
        assert!(cap.is_active());
    }
}
```